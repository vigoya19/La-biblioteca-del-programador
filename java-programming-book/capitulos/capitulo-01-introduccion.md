# Capítulo 1: Introducción a Java

Java no es simplemente otro lenguaje de programación. Es un ecosistema vivo que mueve miles de millones de dólares en la economía digital, corre en más de 3 mil millones de dispositivos y es la columna vertebral de empresas como Netflix, Amazon, Spotify, Uber y la mayoría de los bancos del mundo. Desde que nació en 1995, Java ha demostrado una capacidad de adaptación extraordinaria: sobrevivió a la burbuja de las puntocom, a la guerra de los applets contra JavaScript, al auge de los smartphones, y hoy en 2026 sigue siendo el lenguaje #1 en aplicaciones empresariales.

En este capítulo no solo aprenderás qué es Java y cómo escribir tu primer programa. Te voy a contar **por qué** Java es como es, **qué ocurre realmente** cuando ejecutas `java HolaMundo`, y te daré el **modelo mental** que necesitas para pensar como un programador Java profesional. Siéntate, sírvete un café (Java, obviamente), y empecemos.

---

## 1.1 ¿Qué es Java?

Java es un **lenguaje de programación de propósito general, orientado a objetos, fuertemente tipado y basado en clases**, diseñado por James Gosling y su equipo en Sun Microsystems con una premisa revolucionaria para su época: **"Write Once, Run Anywhere"** (WORA) — "escríbelo una vez, ejecútalo en cualquier lugar".

Pero reducir Java a su definición técnica sería como decir que un Ferrari es "un vehículo de cuatro ruedas con motor de combustión". Java es mucho más que su sintaxis.

### 1.1.1 La filosofía WORA

La idea detrás de WORA es una de las decisiones de ingeniería más brillantes en la historia del software: en lugar de compilar el código fuente directamente a instrucciones de máquina (que atan el programa a un procesador y sistema operativo específico), Java compila a un formato intermedio llamado **bytecode**. Ese bytecode es ejecutado por una capa de software, la **Máquina Virtual de Java (JVM)**, que sí está adaptada para cada plataforma concreta.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        LA PROMESA W.O.R.A.                                   │
│                                                                             │
│   ┌─────────────────────────┐                                               │
│   │  Código fuente           │  HolaMundo.java                              │
│   │  (escribes UNA vez)      │                                               │
│   └───────────┬───────────────┘                                              │
│               │                                                             │
│               │ javac (compilador Java)                                     │
│               ▼                                                             │
│   ┌─────────────────────────┐                                               │
│   │  Bytecode                │  HolaMundo.class  ◄── UN solo archivo        │
│   │  (formato intermedio)    │                       para TODAS las          │
│   └───────────┬───────────────┘                      plataformas             │
│               │                                                             │
│      ┌────────┼────────────────────┐                                        │
│      ▼        ▼                    ▼                                        │
│   ┌──────┐ ┌──────┐ ┌──────┐  ┌──────┐  ┌──────┐                          │
│   │ JVM  │ │ JVM  │ │ JVM  │  │ JVM  │  │ JVM  │                          │
│   │Linux │ │macOS │ │ Win  │  │ARM64 │  │s390x │                          │
│   │x86_64│ │x86_64│ │x86_64│  │(AWS  │  │(IBM  │                          │
│   │      │ │      │ │      │  │Grav.)│  │mainf.)│                          │
│   └──────┘ └──────┘ └──────┘  └──────┘  └──────┘                          │
│        (ejecutas en CUALQUIER lugar)                                        │
└─────────────────────────────────────────────────────────────────────────────┘
```

En lenguajes como C o C++, si escribes un programa en tu portátil Windows, necesitas compilarlo de nuevo (y a menudo modificarlo) para que funcione en un servidor Linux, y otra vez para un Mac. Java eliminó ese problema de raíz.

### 1.1.2 Java en la industria real

Una cosa es hablar de características técnicas y otra muy distinta es saber **dónde se usa Java y por qué te pagan por saberlo**. Aquí tienes el mapa real de Java en la industria:

#### Backend empresarial con Spring Boot

Este es, con diferencia, el principal nicho laboral de Java. **Spring Boot** es un framework que simplifica la creación de aplicaciones web y microservicios. Empresas como Netflix, Amazon, Uber y Alibaba construyen sus backends con Spring Boot. Un controlador REST típico:

```java
@RestController
@RequestMapping("/api/productos")
public class ProductoController {

    @GetMapping
    public List<Producto> listar() {
        return productoService.obtenerTodos();
    }

    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    public Producto crear(@Valid @RequestBody Producto nuevo) {
        return productoService.guardar(nuevo);
    }
}
```

¿Por qué Java para backend? Escalabilidad probada, ecosistema maduro de librerías, monitoreo (Micrometer, Prometheus), transacciones distribuidas, y una cantidad masiva de desarrolladores disponibles.

#### Android

Aunque Kotlin es ahora el lenguaje preferido por Google, **todo el runtime de Android está basado en Java (ART/Dalvik)**. Millones de aplicaciones en Google Play contienen código Java, y las APIs del SDK de Android son APIs Java. Si aprendes Java, aprendes Android automáticamente.

#### Big Data y procesamiento distribuido

**Apache Spark**, **Apache Hadoop**, **Apache Flink**, **Apache Kafka** — todos están escritos en Java o en lenguajes de la JVM. Cuando procesas terabytes de datos en un clúster de cientos de máquinas, es muy probable que Java (o Scala, que corre en la JVM) esté haciendo el trabajo pesado.

```java
// Ejemplo con Apache Spark (Java API)
SparkSession spark = SparkSession.builder()
    .appName("AnalisisVentas")
    .master("local[*]")
    .getOrCreate();

Dataset<Row> ventas = spark.read()
    .option("header", "true")
    .csv("ventas_2024.csv");

ventas.groupBy("region")
    .agg(functions.sum("importe").alias("total"))
    .orderBy(functions.col("total").desc())
    .show();
```

#### Trading financiero de alta frecuencia

Sectores como la banca de inversión y el trading algorítmico usan Java intensivamente. La combinación de **baja latencia** (gracias al JIT), **seguridad de tipos** y **gestión automática de memoria con GC de baja pausa** (ZGC, Shenandoah) hace que Java sea viable incluso para sistemas donde cada microsegundo cuenta. Firmas como Goldman Sachs, JP Morgan y Two Sigma tienen enormes codebases en Java.

#### Videojuegos: Minecraft

El videojuego más vendido de la historia, Minecraft, está escrito íntegramente en Java. Su creador, Markus "Notch" Persson, eligió Java por su facilidad de desarrollo y portabilidad. Minecraft demuestra que Java puede manejar gráficos 3D, física, networking multijugador y un ecosistema masivo de mods, todo sobre la JVM.

#### Dónde NO usar Java (y por qué)

Sé honesto conmigo: Java no es la mejor opción para todo. Apenas se usa en:

| Área                        | Por qué no Java                                      |
|-----------------------------|------------------------------------------------------|
| Frontend web                | JavaScript/TypeScript dominan el navegador           |
| Sistemas embebidos pequeños | El footprint de la JVM es grande; C y Rust mandan    |
| Sistemas operativos         | Java corre sobre un SO, no es el SO                  |
| Machine Learning / IA       | Python con PyTorch/TensorFlow es el estándar         |
| Scripting rápido            | Bash, Python o Node.js son más ágiles                |

Conocer las **limitaciones** de tu herramienta es tan importante como conocer sus fortalezas.

### 1.1.3 El ecosistema de lenguajes de la JVM

Una de las decisiones de diseño más infravaloradas de Java es que la JVM **no está atada a un solo lenguaje**. El bytecode es un formato abierto y documentado, lo que ha permitido que nazcan lenguajes enteros que compilan a bytecode y se ejecutan en la JVM.

```
┌─────────────────────────────────────────────────────────────────┐
│                    ECOSISTEMA JVM                                │
│                                                                 │
│   ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐          │
│   │   Java   │ │  Kotlin  │ │  Scala   │ │  Groovy  │  ...     │
│   │ .java    │ │ .kt      │ │ .scala   │ │ .groovy  │          │
│   └────┬─────┘ └────┬─────┘ └────┬─────┘ └────┬─────┘          │
│        │            │            │            │                 │
│        ▼            ▼            ▼            ▼                 │
│   ┌─────────────────────────────────────────────────┐          │
│   │               BYTECODE (.class)                 │          │
│   │      Todos compilan al mismo formato            │          │
│   └──────────────────────┬──────────────────────────┘          │
│                          ▼                                     │
│   ┌─────────────────────────────────────────────────┐          │
│   │              JVM HOTSPOT                         │          │
│   │  ClassLoader → JIT Compiler → Garbage Collector │          │
│   └──────────────────────┬──────────────────────────┘          │
│                          ▼                                     │
│                    SISTEMA OPERATIVO                            │
└─────────────────────────────────────────────────────────────────┘
```

**Kotlin**: Creado por JetBrains en 2011. Es el lenguaje preferido de Android. Ofrece null-safety a nivel de tipos, extension functions, corrutinas y data classes. 100% interoperable con Java.

```kotlin
data class Usuario(val nombre: String, val edad: Int)

fun main() {
    val usuarios = listOf(Usuario("Ana", 28), Usuario("Luis", 35))
    val mayoresDe30 = usuarios.filter { it.edad > 30 }
    println(mayoresDe30)  // [Usuario(nombre=Luis, edad=35)]
}
```

**Scala**: Creado por Martin Odersky en 2004. Combina OOP y programación funcional. Base de Apache Spark, Akka y Kafka. Su sistema de tipos es uno de los más potentes sobre la JVM.

**Groovy**: Lenguaje dinámico con sintaxis similar a Java pero más concisa. Es el lenguaje de los scripts de Gradle. También usado en testing con Spock.

**Clojure**: Dialecto de Lisp en la JVM. Inmutabilidad por defecto y concurrencia con STM. Usado por Walmart y Nubank.

La belleza de este ecosistema: si aprendes la JVM, entiendes **todos estos lenguajes**. Y todos comparten las mismas librerías. Una librería Java se usa desde Kotlin; una de Scala se usa desde Java. Esto es interoperabilidad real.

### 1.1.4 Características principales de Java

- **Orientado a objetos**: Todo en Java (excepto los 8 tipos primitivos: `int`, `long`, `double`, `float`, `boolean`, `char`, `byte`, `short`) es un objeto. Java organiza el código en clases que encapsulan datos y comportamientos.

- **Fuertemente tipado y estático**: Cada variable tiene un tipo definido en tiempo de compilación. El compilador rechaza asignaciones incorrectas antes de ejecutar.

```java
int edad = "veinticinco";  // ERROR de compilación
```

- **Independiente de la plataforma**: WORA.

- **Robusto**: Sin `malloc`/`free`, sin punteros, verificación de tipos exhaustiva, excepciones chequeadas.

- **Seguro**: Bytecode verifier, sandbox de ejecución, sin acceso directo a memoria.

- **Multihilo**: `Thread`, `synchronized`, `java.util.concurrent`. Concurrencia nativa desde el día 1.

- **Alto rendimiento**: HotSpot JIT compila a código nativo en runtime con rendimiento comparable a C++ en cargas de servidor.

---

## 1.2 Historia y evolución

### 1.2.1 Los orígenes olvidados

En 1991, Sun Microsystems formó un equipo secreto llamado **"Green Team"**, liderado por James Gosling, Mike Sheridan y Patrick Naughton. Su misión: crear un lenguaje para la siguiente ola de dispositivos electrónicos inteligentes.

Gosling creó **"Oak"** (roble), inspirado por el árbol frente a su oficina. Oak era independiente de la plataforma — algo revolucionario en una época donde cada dispositivo tenía su propio procesador.

El Green Team construyó el ***7 (Star Seven)**, un control remoto interactivo con pantalla táctil. Lo presentaron a empresas de TV por cable... y nadie lo compró. La televisión interactiva no existía. Oak estuvo a punto de desaparecer.

```
1991 ────► 1992 ────► 1993 ────► 1994 ────► 1995
 │            │           │           │           │
Green        *7         La TV       Nace el     Java 1.0
Team       presentado  interactiva  navegador   presentado
crea Oak   a empresas  no despega   Mosaic      en SunWorld
```

### 1.2.2 El renacimiento: Java y la Web

En 1993 apareció Mosaic, el primer navegador gráfico. La Web explotó. Bill Joy, co-fundador de Sun, tuvo una visión: el lenguaje creado para electrodomésticos era **perfecto para la Web**. Portable, seguro y podía incrustarse en páginas.

En 1995, renombraron Oak a **Java** (en honor al café de la isla de Java) y lanzaron el navegador **HotJava**, que ejecutaba **applets** dentro de páginas web. El 23 de mayo de 1995, en SunWorld, la demo fue electrizante: animación 3D interactiva dentro de una página web. Marc Andreessen anunció que Netscape Navigator soportaría Java. El resto es historia.

### 1.2.3 Por qué Java ganó (comparación con C++)

Para entender el éxito de Java, hay que entender el contexto de 1995. **C++ dominaba**, pero era difícil y peligroso:

```
┌──────────────────────┬──────────────────────────────────────┐
│      C++ (1995)      │            Java (1995)               │
├──────────────────────┼──────────────────────────────────────┤
│ Punteros y acceso    │ No hay punteros: referencias         │
│ directo a memoria    │ seguras gestionadas por la JVM       │
├──────────────────────┼──────────────────────────────────────┤
│ malloc() / free()    │ Garbage Collector automático         │
│ Manual. Causa #1 de  │ Adiós a memory leaks y               │
│ bugs y crashes       │ double-free                         │
├──────────────────────┼──────────────────────────────────────┤
│ Compila a código     │ Compila a bytecode portable          │
│ nativo por SO.       │ Mismo .class en Windows, Linux,      │
│ Recompilar siempre   │ macOS sin cambios                    │
├──────────────────────┼──────────────────────────────────────┤
│ Sin verificación     │ Bytecode Verifier + sandbox de       │
│ de seguridad         │ seguridad                            │
├──────────────────────┼──────────────────────────────────────┤
│ Herencia múltiple    │ Herencia simple + múltiples          │
│ compleja (diamond    │ interfaces. Más limpio y seguro      │
│ problem)             │                                      │
├──────────────────────┼──────────────────────────────────────┤
│ Librería estándar    │ API estándar extensa desde día 1:    │
│ mínima               │ redes, GUI, hilos, colecciones...    │
├──────────────────────┼──────────────────────────────────────┤
│ Hilos sin soporte    │ Threading nativo integrado desde     │
│ en el estándar       │ la versión 1.0                       │
└──────────────────────┴──────────────────────────────────────┘
```

Java eliminó el 80% de los dolores de cabeza de C++. Los programadores migraron en masa.

### 1.2.4 La guerra de los applets vs JavaScript

En 1995, Netscape le pidió a Brendan Eich que creara un "lenguaje de scripting para el navegador" en **10 días**. Creó Mocha → LiveScript → **JavaScript** (marketing para aprovechar el hype de Java, aunque no tienen nada que ver técnicamente).

Los **applets de Java** y JavaScript compitieron por la interactividad web. Los applets eran más potentes, pero:
- Requerían un plugin JRE lento de cargar
- La integración con el DOM era torpe
- Cada actualización de seguridad era problemática

JavaScript, aunque limitado, estaba **integrado** en el navegador, cargaba instantáneamente y se incrustaba en HTML. Para 2010, los applets estaban muertos. JavaScript ganó la guerra del navegador.

**Pero Java ganó la guerra del servidor.** Mientras JavaScript luchaba en el frontend, Java conquistó los centros de datos.

### 1.2.5 Por qué Java sigue siendo el #1 en enterprise

1. **Estabilidad de APIs**: Código Java de 2005 compila en Java 21 con mínimas modificaciones. Las empresas valoran estabilidad sobre novedad.

2. **Ecosistema masivo**: Millones de librerías en Maven Central. Si necesitas hacer algo, ya existe una librería madura.

3. **Talento abundante**: Millones de programadores Java. Contratar es fácil.

4. **Monitoreo y observabilidad**: JMX, Java Flight Recorder, agentes. Ningún otro runtime ofrece este nivel de introspección sin modificar código.

5. **Rendimiento predecible**: Aunque Go y Rust ganan en microbenchmarks, la JVM con HotSpot produce rendimiento consistente bajo cargas sostenidas.

6. **Gobernanza compartida**: El Java Community Process (JCP) garantiza que ninguna empresa controle el lenguaje. Oracle, Red Hat, IBM, Microsoft, Google — todos tienen voz.

### 1.2.6 La era Sun Microsystems (1995-2010)

- **1996 (JDK 1.0)**: 212 clases en 8 paquetes. Applets y AWT.
- **1997 (JDK 1.1)**: JDBC, JavaBeans, RMI, clases internas.
- **1998 (Java 1.2 / "Java 2")**: J2SE/J2EE/J2ME. Swing. Collections Framework.
- **2000 (Java 1.3)**: HotSpot JVM.
- **2002 (Java 1.4)**: Aserciones, regex, logging, NIO.
- **2004 (Java 5)**: **REVOLUCIÓN #1**: Genéricos, anotaciones, enums, autoboxing, for-each, varargs.
- **2006 (Java 6)**: Mejoras de rendimiento. OpenJDK (GPL).

### 1.2.7 La era Oracle (2010-presente)

- **2011 (Java 7)**: Try-with-resources, switch con String, diamond operator, NIO.2.
- **2014 (Java 8)**: **REVOLUCIÓN #2**: Lambdas, Stream API, `java.time`, Optional, métodos default.
- **2017 (Java 9)**: Módulos (Project Jigsaw), JShell, métodos factory para colecciones.
- **2018 (Java 10-11)**: Ciclo de releases cada 6 meses. `var`. Java 11: primera **LTS** moderna.
- **2019-2020 (Java 12-15)**: Switch expressions, text blocks, records (preview), ZGC y Shenandoah.
- **2021 (Java 17 LTS)**: Records, sealed classes, pattern matching para switch.
- **2023 (Java 21 LTS)**: **Virtual Threads** (Project Loom), pattern matching para records, sequenced collections.

### 1.2.8 Tabla de versiones LTS comparadas

| Característica        | Java 8 | Java 11 | Java 17 | Java 21 |
|-----------------------|:------:|:-------:|:-------:|:-------:|
| Lambdas + Streams     |   Sí   |   Sí    |   Sí    |   Sí    |
| Módulos (JPMS)        |   No   |   Sí    |   Sí    |   Sí    |
| `var` local           |   No   |   Sí    |   Sí    |   Sí    |
| Text blocks           |   No   |   No    |   Sí    |   Sí    |
| Records               |   No   |   No    |   Sí    |   Sí    |
| Switch expressions    |   No   |   No    |   Sí    |   Sí    |
| Pattern matching      |   No   |   No    |   Sí    |   Sí    |
| Sealed classes        |   No   |   No    |   Sí    |   Sí    |
| Virtual Threads       |   No   |   No    |   No    |   Sí    |
| Fin soporte           |  2030  |  2027   |  2029   |  2031   |

---

## 1.3 JVM, JRE, JDK: El corazón de Java

### 1.3.1 Los tres componentes

```
┌─────────────────────────────────────────────────────────────────────────┐
│                              JDK                                         │
│  (Java Development Kit - lo que instalas para desarrollar)               │
│                                                                         │
│  ┌───────────────────────────────────────────────────────────────────┐  │
│  │                           JRE                                      │  │
│  │  (Java Runtime Environment - necesario para ejecutar)              │  │
│  │                                                                   │  │
│  │  ┌─────────────────────────────────────────────────────────────┐  │  │
│  │  │                        JVM                                   │  │  │
│  │  │  (Java Virtual Machine - el motor de ejecución)              │  │  │
│  │  │                                                             │  │  │
│  │  │  ┌─────────────┐  ┌──────────────┐  ┌──────────────────┐   │  │  │
│  │  │  │ ClassLoader │  │ Bytecode     │  │ Execution Engine │   │  │  │
│  │  │  │ Subsystem   │──│ Verifier     │──│ (Interpreter +   │   │  │  │
│  │  │  │ (Bootstrap, │  │ (seguridad)  │  │  JIT Compiler)   │   │  │  │
│  │  │  │  Platform,  │  └──────────────┘  └────────┬─────────┘   │  │  │
│  │  │  │  Application│                             │             │  │  │
│  │  │  └─────────────┘                    ┌─────────▼──────────┐  │  │  │
│  │  │                                     │ Garbage Collector  │  │  │  │
│  │  │  ┌──────────────────────────────┐   │ (Serial/Parallel/  │  │  │  │
│  │  │  │ Áreas de Memoria en Runtime  │   │  G1/ZGC/Shenandoah)│  │  │  │
│  │  │  │ Heap │ Stack │ Metaspace │...│   └────────────────────┘  │  │  │
│  │  │  └──────────────────────────────┘                           │  │  │
│  │  └─────────────────────────────────────────────────────────────┘  │  │
│  │                                                                   │  │
│  │  ┌────────────────────────────────────────────────────────────┐   │  │
│  │  │  Bibliotecas estándar (API de Java)                       │   │  │
│  │  │  java.lang │ java.util │ java.io │ java.net │ java.sql │...│   │  │
│  │  └────────────────────────────────────────────────────────────┘   │  │
│  └───────────────────────────────────────────────────────────────────┘  │
│                                                                         │
│  ┌──────────────────────────────────────────────────────────────────┐   │
│  │  Herramientas: javac │ java │ jar │ javadoc │ jshell │ jlink     │   │
│  └──────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────┘
```

- **JVM**: El motor que ejecuta bytecode. Especificación con múltiples implementaciones (HotSpot, OpenJ9, GraalVM).
- **JRE**: JVM + bibliotecas estándar. Lo mínimo para ejecutar. Desde Java 11 se usa `jlink` para crear runtimes personalizados.
- **JDK**: JRE + compilador + herramientas. Lo que instalas para desarrollar.

### 1.3.2 Cómo la JVM carga clases: el sistema ClassLoader

Cuando ejecutas `java HolaMundo`, la JVM carga las clases mediante un sistema jerárquico con tres principios:

1. **Delegación**: Un ClassLoader primero pregunta a su padre.
2. **Visibilidad**: Un hijo ve las clases del padre, no al revés.
3. **Unicidad**: Una clase se carga una sola vez en la jerarquía.

```
┌───────────────────────────────────────────────────────────────────┐
│                JERARQUÍA DE CLASSLOADERS                           │
│                                                                   │
│   ┌─────────────────────────────────────────────────────────┐     │
│   │  Bootstrap ClassLoader (nativo, C++)                     │     │
│   │  Carga: java.lang.*, java.util.*, java.io.*, ...        │     │
│   │  Desde: <JAVA_HOME>/lib/modules                         │     │
│   │  No tiene padre. getClassLoader() devuelve null.        │     │
│   └──────────────────────────┬──────────────────────────────┘     │
│                              │ "padre de"                          │
│                              ▼                                     │
│   ┌─────────────────────────────────────────────────────────┐     │
│   │  Platform ClassLoader (antes Extension ClassLoader)      │     │
│   │  Carga: APIs de la plataforma                           │     │
│   │  Desde: <JAVA_HOME>/lib                                 │     │
│   └──────────────────────────┬──────────────────────────────┘     │
│                              │ "padre de"                          │
│                              ▼                                     │
│   ┌─────────────────────────────────────────────────────────┐     │
│   │  Application ClassLoader (System ClassLoader)            │     │
│   │  Carga: tus clases, las del classpath (-cp)              │     │
│   └─────────────────────────────────────────────────────────┘     │
└───────────────────────────────────────────────────────────────────┘
```

**¿Por qué importa?** El infame `ClassNotFoundException` o `NoClassDefFoundError` — el 99% de las veces es classpath mal configurado. Frameworks como Spring y Tomcat usan ClassLoaders personalizados para aislar aplicaciones.

Ejemplo para ver los ClassLoaders:

```java
public class VerClassLoaders {
    public static void main(String[] args) {
        // Clase propia → Application ClassLoader
        System.out.println("Mi clase: " +
            VerClassLoaders.class.getClassLoader());

        // Clase del JDK → Bootstrap (null = nativo C++)
        System.out.println("String:  " +
            String.class.getClassLoader());   // null

        System.out.println("Platform CL: " +
            ClassLoader.getPlatformClassLoader());
        System.out.println("System CL:   " +
            ClassLoader.getSystemClassLoader());
    }
}
```

### 1.3.3 Bytecode: el lenguaje que la JVM ejecuta

`javac` genera bytecode, no código máquina. Veamos un ejemplo:

```java
public class Suma {
    public static int sumar(int a, int b) {
        return a + b;
    }
}
```

Compilamos y usamos `javap` para inspeccionar:

```bash
javac Suma.java
javap -c Suma
```

Salida:

```
Compiled from "Suma.java"
public class Suma {
  public Suma();
    Code:
       0: aload_0
       1: invokespecial #1    // Method java/lang/Object."<init>":()V
       4: return

  public static int sumar(int, int);
    Code:
       0: iload_0             // Carga el primer parámetro (a)
       1: iload_1             // Carga el segundo parámetro (b)
       2: iadd                // Suma de enteros
       3: ireturn             // Retorna el resultado
}
```

El bytecode es una máquina de pila (stack-based). Así se ejecuta `sumar(5, 3)`:

```
Inicial:          Paso 0: iload_0   Paso 1: iload_1   Paso 2: iadd     Paso 3: ireturn
├────────┤        ├────────┤         ├────────┤         ├────────┤        ├────────┤
│ (vacía)│        │   5    │         │   3    │         │   8    │        │ (vacía)│
├────────┤        ├────────┤         │   5    │         ├────────┤        ├────────┤
                                     ├────────┤
```

**¿Por qué importa el bytecode?** Depuración avanzada, comprensión del rendimiento, y poliglotismo JVM (Kotlin, Scala, Groovy compilan al mismo bytecode).

Opciones útiles de `javap`:

```bash
javap -c        Suma    # Bytecode con instrucciones
javap -v        Suma    # Información verbose (constant pool)
javap -p        Suma    # Miembros privados
javap -l        Suma    # Tabla de números de línea
javap -c -p -v  Suma    # Todo junto
```

### 1.3.4 JIT: Cuando la JVM se convierte en compilador

Al principio, Java interpretaba bytecode → lento. **HotSpot** cambió todo con la **compilación Just-In-Time (JIT)**.

#### El principio: "caliente" vs "frío"

En cualquier programa, el 80-90% del tiempo se gasta en el 10-20% del código (los *hot spots*).

```
┌──────────────────────────────────────────────────────────────────────┐
│           CICLO DE VIDA DE UN MÉTODO EN HOTSPOT                      │
│                                                                      │
│   [Bytecode .class]                                                  │
│       │                                                             │
│       ▼                                                             │
│   ┌──────────┐     ¿suficientes    ┌──────────────┐                 │
│   │Intérprete│──── invocaciones?──►│ Compilación  │                 │
│   │(perfilado│       (umbral)      │    JIT       │                 │
│   │   lento) │                    │  (convierte  │                 │
│   └──────────┘                    │  a nativo)   │                 │
│       │                           └──────┬───────┘                 │
│       │ (frío: sigue                    │                          │
│       │  interpretándose)               ▼                          │
│       │                         ┌────────────────┐                  │
│       │                         │ Código nativo  │  ¡Velocidad     │
│       │                         │ en caché       │  máxima!        │
│       │                         └────────────────┘                  │
│       │                                                             │
│       └─────────► Desoptimización ◄──────────────────────────────── │
│           (si las condiciones cambian, revierte a interpretado)     │
└──────────────────────────────────────────────────────────────────────┘
```

#### C1 y C2: Dos compiladores, dos filosofías

| Característica            | C1 (Client Compiler)          | C2 (Server Compiler)              |
|---------------------------|-------------------------------|-----------------------------------|
| Velocidad de compilación  | Rápida (microsegundos)        | Lenta (cientos de ms)             |
| Optimizaciones            | Ligeras                       | Agresivas (inlineación, escape analysis, loop unrolling) |
| Uso de CPU al compilar    | Bajo                          | Alto                              |
| Ideal para                | Arranque rápido, desktop      | Rendimiento sostenido, servidor   |

#### Compilación por niveles (Tiered Compilation)

Desde Java 8, el modo por defecto combina C1 y C2 en 5 niveles:

```
Nivel 0: Interpretación pura (perfilado)
   │     ↓ umbral alcanzado
Nivel 1: C1 sin perfilado (rápido)
   │     ↓ más caliente
Nivel 2: C1 con perfilado limitado
   │     ↓ más caliente aún
Nivel 3: C1 con perfilado completo
   │     ↓ merece optimización agresiva
Nivel 4: C2 con todas las optimizaciones (máxima calidad)
```

Flags para experimentar con los niveles:

```bash
java -XX:-TieredCompilation MiApp        # Sin tiered
java -XX:TieredStopAtLevel=1 MiApp       # Solo C1 (arranque rápido)
java -XX:TieredStopAtLevel=4 MiApp       # Niveles completos (default)
```

#### Ver qué se está compilando

```bash
java -XX:+PrintCompilation MiApp
```

Salida típica:

```
     68    1       3       java.lang.String::hashCode (55 bytes)
     69    2       3       java.lang.String::charAt (29 bytes)
     70    3       4       java.lang.Object::<init> (1 bytes)
     73    6     n 0       java.lang.System::arraycopy (native)
    120   15       4       com.miempresa.Reporte::generar (247 bytes)
```

Columnas: timestamp | id_compilación | atributos | nivel | método (tamaño).

Atributos: `%` = on-stack replacement, `s` = synchronized, `!` = tiene excepciones, `n` = nativo.

```bash
# Ver qué métodos fueron inlineados por C2 (muy detallado)
java -XX:+UnlockDiagnosticVMOptions -XX:+PrintInlining MiApp
```

### 1.3.5 Garbage Collection en profundidad

En C: `malloc` / `free` manual → memory leaks, double-free, use-after-free. Java eliminó esto con el **Garbage Collector (GC)**: identifica objetos no alcanzables y libera su memoria automáticamente.

#### Anatomía del Heap

```
┌───────────────────────────────────────────────────────────────────────────┐
│                    HEAP (memoria dinámica para objetos)                     │
│                                                                           │
│  ┌─────────────────────────────────┐  ┌────────────────────────────────┐  │
│  │      YOUNG GENERATION           │  │      OLD GENERATION            │  │
│  │      (Generación Joven)         │  │      (Generación Vieja)        │  │
│  │                                 │  │                                │  │
│  │  ┌────────┐┌────────┐┌───────┐ │  │  ┌──────────────────────────┐  │  │
│  │  │  EDEN  ││SURVIVOR││SURVIVOR│ │  │  │                          │  │  │
│  │  │ (nuevos││   0    ││   1   │ │  │  │   Objetos longevos        │  │  │
│  │  │objetos)││  (S0)  ││  (S1) │─│──│──│   (cachés, singletons)    │  │  │
│  │  └────────┘└────────┘└───────┘ │  │  │                          │  │  │
│  └─────────────────────────────────┘  └────────────────────────────────┘  │
│                                                                           │
│  ┌──────────────────────────────────────────────────────────────────┐     │
│  │                        METASPACE                                  │     │
│  │  Metadatos de clases, constant pool, variables estáticas          │     │
│  │  No es parte del heap. Crece automáticamente.                     │     │
│  │  -XX:MaxMetaspaceSize=256m para limitarlo.                        │     │
│  └──────────────────────────────────────────────────────────────────┘     │
└───────────────────────────────────────────────────────────────────────────┘
```

**Ciclo de vida de un objeto:**

1. **Nace** en Eden (`new`)
2. **Sobrevive** a un Minor GC → se copia a Survivor (S0/S1 alternando)
3. Tras varias supervivencias (umbral de tenuring, configurable con `-XX:MaxTenuringThreshold`) → **promociona** a Old Generation
4. En Old, eventualmente un Major GC lo recolecta si ya no es referenciado

#### Algoritmos de GC: cuál usar

```
┌──────────┬───────────┬───────────┬──────────────┬─────────────┬───────────────┐
│          │  Serial   │ Parallel  │      G1      │     ZGC     │  Shenandoah   │
│          │           │(Throughput)│   (default)  │             │               │
├──────────┼───────────┼───────────┼──────────────┼─────────────┼───────────────┤
│ Hilos GC │    1      │ Múltiples │  Múltiples   │  Múltiples  │  Múltiples    │
├──────────┼───────────┼───────────┼──────────────┼─────────────┼───────────────┤
│ Pausa    │  Larga    │  Larga    │  < 200 ms    │  < 1 ms     │  < 10 ms      │
│ STW      │(segundos) │(segundos) │  (objetivo)  │  (sub-ms)   │               │
├──────────┼───────────┼───────────┼──────────────┼─────────────┼───────────────┤
│ Heap máx │  Pequeño  │  Grande   │  Grande      │  Masivo (TB)│  Grande       │
├──────────┼───────────┼───────────┼──────────────┼─────────────┼───────────────┤
│ Ideal    │  Apps     │  Batch /  │  Servidores  │  Ultra-baja │  Similar a    │
│ para     │  simples  │  backend │  web, APIs   │  latencia   │  ZGC          │
├──────────┼───────────┼───────────┼──────────────┼─────────────┼───────────────┤
│ Desde    │   1.0     │   1.3     │     7        │     15      │     12        │
│ Java     │           │           │(default 9+)  │(prod. 15)   │(prod. 15)     │
└──────────┴───────────┴───────────┴──────────────┴─────────────┴───────────────┘
```

**Regla práctica para elegir:**

| Situación | GC recomendado |
|-----------|---------------|
| Heap < 4GB, no importan pausas | **Parallel GC** (máximo throughput) |
| Servidor web/API, heap 4-64GB | **G1** (default desde Java 9) |
| Pausas < 10ms en heaps grandes | **ZGC** o **Shenandoah** |
| Contenedor Docker 1 CPU, 512MB | **Serial GC** (menor overhead) |
| Aplicación batch, procesamiento nocturno | **Parallel GC** |

#### Flags importantes de configuración

```bash
# Tamaño del heap
-Xms512m                        # Tamaño inicial del heap
-Xmx4g                          # Tamaño máximo del heap
# Recomendación: -Xms == -Xmx en producción para evitar redimensionamientos

# Seleccionar GC
-XX:+UseSerialGC                # Serial
-XX:+UseParallelGC              # Parallel (throughput)
-XX:+UseG1GC                    # G1 (default desde Java 9)
-XX:+UseZGC                     # ZGC (Java 15+)
-XX:+UseShenandoahGC            # Shenandoah

# G1 específico
-XX:MaxGCPauseMillis=200        # Objetivo de pausa máxima en ms (default 200)
-XX:G1HeapRegionSize=16m        # Tamaño de región G1 (potencia de 2, 1-32MB)
-XX:InitiatingHeapOccupancyPercent=45  # % de ocupación que dispara ciclo concurrente

# Logging de GC (unificado desde Java 9)
-Xlog:gc*:file=gc.log:time,level,tags   # Log detallado a archivo
-Xlog:gc:stdout:time            # Log básico a salida estándar

# Metaspace
-XX:MaxMetaspaceSize=256m       # Límite para metadatos de clases

# Ejemplo completo para un servidor en producción
java -Xms4g -Xmx4g \
     -XX:+UseG1GC \
     -XX:MaxGCPauseMillis=200 \
     -XX:MaxMetaspaceSize=256m \
     -Xlog:gc*:file=/var/log/mi-app/gc.log:time,level,tags:filecount=10,filesize=100M \
     -jar mi-app.jar
```

#### Cómo monitorizar el GC con jstat

```bash
# Primero encuentra el PID del proceso Java
jps -l

# Estadísticas de GC cada 1000ms, 10 muestras
jstat -gc <PID> 1000 10
```

Salida típica (columnas principales):

```
 S0C    S1C    S0U    S1U      EC       EU        OC         OU       MC     MU
 0.0   512.0   0.0   512.0  128512.0  64000.0   245760.0   122880.0  44800.0 43210.5
```

Leyenda de columnas:
- **S0C/S1C**: Capacidad de Survivor 0/1 en KB. **S0U/S1U**: Uso actual
- **EC/EU**: Capacidad y uso de Eden
- **OC/OU**: Capacidad y uso de Old Generation
- **MC/MU**: Capacidad y uso de Metaspace

Para una vista más completa con conteos de GC:

```bash
jstat -gcutil <PID> 1000
```

```
  S0     S1     E      O      M     CCS    YGC     YGCT    FGC    FGCT     GCT
  0.00  100.0  49.81  50.00  96.43  92.15    42    0.823     0    0.000    0.823
```

Columnas clave:
- **YGC / YGCT**: Número y tiempo total de Young GC
- **FGC / FGCT**: Número y tiempo total de Full GC — **¡si FGC crece rápido, tienes un problema!** Probablemente memory leak o heap demasiado pequeño.
- **GCT**: Tiempo total acumulado en GC desde el arranque

---

## 1.4 Instalación del JDK

Instalar Java correctamente es tu primera responsabilidad como desarrollador. No uses el Java preinstalado en tu sistema — probablemente sea antiguo. Instala una distribución moderna y aprende a gestionar múltiples versiones.

### 1.4.1 ¿Qué distribución elegir?

Desde que Oracle empezó a cobrar licencias por Oracle JDK para uso comercial (2019), las builds de OpenJDK son el estándar de facto. Todas comparten el mismo código base, pero difieren en soporte y pruebas.

| Distribución | Mantenedor | Licencia | Recomendación |
|---|---|---|---|
| **Eclipse Temurin** | Eclipse Foundation (Adoptium) | Gratuita | **Nuestra recomendación** |
| Amazon Corretto | Amazon | Gratuita | Si despliegas en AWS |
| Azul Zulu | Azul Systems | Gratuita (soporte pago) | Soporte SLA |
| Oracle OpenJDK | Oracle | Gratuita | Build de referencia |
| GraalVM | Oracle Labs | Community gratis | Alto rendimiento, polyglot |

Usaremos **Eclipse Temurin (Adoptium)**: la distribución más descargada, con builds para todas las plataformas y actualizaciones de seguridad trimestrales.

### 1.4.2 SDKMAN! — La mejor forma en Linux/macOS

**SDKMAN!** (Software Development Kit Manager) permite instalar, cambiar y listar múltiples versiones de Java y otras herramientas JVM con comandos simples:

```bash
# Instalar SDKMAN! (un solo comando)
curl -s "https://get.sdkman.io" | bash
source "$HOME/.sdkman/bin/sdkman-init.sh"

# Listar versiones disponibles
sdk list java

# Instalar la última LTS de Temurin
sdk install java 21.0.1-tem

# Instalar varias versiones en paralelo
sdk install java 17.0.9-tem
sdk install java 11.0.21-tem

# Cambiar a otra versión (solo para esta terminal)
sdk use java 17.0.9-tem

# Establecer una versión por defecto (persistente)
sdk default java 21.0.1-tem

# Ver qué versión está activa
sdk current java

# Desinstalar una versión que ya no necesitas
sdk uninstall java 11.0.21-tem
```

Con SDKMAN!, cambiar entre versiones según el proyecto es trivial:

```bash
cd ~/proyectos/mi-api-nueva
sdk use java 21.0.1-tem
./mvnw spring-boot:run

cd ~/proyectos/legacy-app
sdk use java 17.0.9-tem
./mvnw spring-boot:run
```

### 1.4.3 Instalación por sistema operativo

#### Windows

1. Ve a [https://adoptium.net/download/](https://adoptium.net/download/)
2. Selecciona **Java 21 LTS**, Windows, **x64**
3. Descarga el instalador `.msi`
4. Ejecuta. **Marca "Set JAVA_HOME variable"** y "Add to PATH"
5. Verifica en PowerShell o CMD:

```batch
java -version
javac -version
```

Si `java` no se reconoce después de instalar:
1. Busca "Variables de entorno" → "Editar las variables de entorno del sistema"
2. Crea `JAVA_HOME` = `C:\Program Files\Eclipse Adoptium\jdk-21.0.X.X-hotspot\`
3. Edita `Path` y añade `%JAVA_HOME%\bin`
4. Reinicia la terminal y verifica

#### macOS

**Opción A: SDKMAN! (recomendada — ver sección 1.4.2)**

**Opción B: Homebrew**

```bash
brew install --cask temurin@21
```

**Opción C: Instalador `.pkg`**

Descarga desde adoptium.net, ejecuta el `.pkg`. El instalador configura `JAVA_HOME` automáticamente en `/Library/Java/JavaVirtualMachines/`.

Ver dónde se instaló:

```bash
/usr/libexec/java_home -V
```

#### Linux (Ubuntu/Debian)

```bash
sudo apt update && sudo apt install -y wget apt-transport-https
wget -O - https://packages.adoptium.net/artifactory/api/gpg/key/public | \
  sudo tee /usr/share/keyrings/adoptium.asc

echo "deb [signed-by=/usr/share/keyrings/adoptium.asc] \
  https://packages.adoptium.net/artifactory/deb \
  $(awk -F= '/^UBUNTU_CODENAME/{print$2}' /etc/os-release) main" | \
  sudo tee /etc/apt/sources.list.d/adoptium.list

sudo apt update && sudo apt install temurin-21-jdk
```

### 1.4.4 Verificación detallada

Ejecuta `java -version` y analicemos la salida:

```
openjdk version "21.0.1" 2023-10-17 LTS
OpenJDK Runtime Environment Temurin-21.0.1+12 (build 21.0.1+12-LTS)
OpenJDK 64-Bit Server VM Temurin-21.0.1+12 (build 21.0.1+12-LTS, mixed mode, sharing)
```

| Elemento | Significado |
|---|---|
| `openjdk version "21.0.1"` | Versión del JDK |
| `2023-10-17` | Fecha de la build |
| `LTS` | Long-Term Support: actualizaciones de seguridad por años |
| `Temurin-21.0.1+12` | Distribución Eclipse Temurin, build #12 |
| `64-Bit Server VM` | JVM en modo servidor de 64 bits |
| `mixed mode` | Interpretado + JIT compilado (no solo interpretado) |
| `sharing` | Class Data Sharing activo: arranque más rápido |

Verifica también el compilador:

```bash
javac -version       # javac 21.0.1
jar --version        # jar 21.0.1
jshell --version     # jshell 21.0.1
```

### 1.4.5 `JAVA_HOME` y `PATH`

Estas dos variables de entorno causan el 40% de los problemas de configuración en Java.

- **`JAVA_HOME`**: Directorio raíz del JDK. Maven, Gradle, Tomcat e IDEs lo usan para encontrar Java.
- **`PATH`**: Debe incluir `$JAVA_HOME/bin` (o `%JAVA_HOME%\bin` en Windows) para que `java`, `javac`, etc. funcionen desde cualquier terminal.

Cómo verificarlo:

```bash
# Linux / macOS
echo $JAVA_HOME
echo $PATH | tr ':' '\n' | grep java
which java

# Windows (PowerShell)
echo $env:JAVA_HOME
Get-Command java | Select-Object Source
```

Configuración manual en `.bashrc` / `.zshrc`:

```bash
export JAVA_HOME=/usr/lib/jvm/temurin-21-jdk
export PATH=$JAVA_HOME/bin:$PATH
```

**Importante:** Si usas SDKMAN!, **NO configures `JAVA_HOME` manualmente**. SDKMAN! lo gestiona automáticamente.

### 1.4.6 Múltiples versiones en la misma máquina

| Estrategia | Plataforma | Recomendación |
|---|---|---|
| **SDKMAN!** | Linux, macOS, WSL | La mejor opción |
| **jEnv** | Solo macOS | Alternativa |
| **Manual** (alias) | Todas | Plan B |

Estrategia manual con alias (Plan B):

```bash
# En tu .bashrc o .zshrc
alias java21='export JAVA_HOME=/usr/lib/jvm/temurin-21 && export PATH=$JAVA_HOME/bin:$PATH'
alias java17='export JAVA_HOME=/usr/lib/jvm/temurin-17 && export PATH=$JAVA_HOME/bin:$PATH'

# Uso:
java21 && java -version   # → 21.0.1
java17 && java -version   # → 17.0.9
```

---

## 1.5 Tu primer programa: ¡Hola, Mundo!

### 1.5.1 La versión clásica

Crea un archivo `HolaMundo.java` con este contenido exacto:

```java
/**
 * HolaMundo.java
 * Mi primer programa en Java.
 *
 * @author Tu Nombre
 * @version 1.0
 */
public class HolaMundo {
    public static void main(String[] args) {
        System.out.println("¡Hola, Mundo!");
        System.out.println("Bienvenido a Java.");
    }
}
```

### 1.5.2 Explicación línea por línea

```java
/**
 * HolaMundo.java
 * Mi primer programa en Java.
 */
```

**Comentario Javadoc.** `/** ... */` lo procesa la herramienta `javadoc` para generar documentación HTML automática. Las etiquetas `@author` y `@version` son tags estándar.

Java tiene tres tipos de comentarios:
- `//` — Una línea. Ideal para notas breves.
- `/* ... */` — Multilínea. Para deshabilitar bloques temporalmente.
- `/** ... */` — Javadoc. Para documentar clases y métodos.

```java
public class HolaMundo {
```

- **`public`**: Modificador de acceso. Visible desde cualquier otra clase. Un archivo `.java` puede tener solo **una** clase pública.
- **`class`**: Palabra clave que declara una clase. En Java, **todo** el código ejecutable reside dentro de clases.
- **`HolaMundo`**: Nombre en **PascalCase**. **Debe coincidir exactamente con el nombre del archivo** (incluyendo mayúsculas/minúsculas). `HolaMundo.java` debe contener `public class HolaMundo`.

```java
    public static void main(String[] args) {
```

El **punto de entrada**. La JVM busca exactamente esta firma:

| Token | Significado profundo |
|---|---|
| `public` | La JVM necesita acceso sin restricciones para invocar este método |
| `static` | Pertenece a la clase, no a una instancia. La JVM lo llama sin crear objetos |
| `void` | No devuelve valor. El resultado se comunica mediante el **exit code** |
| `main` | Nombre exacto que busca la JVM. Distingue mayúsculas: `Main` **no funciona** |
| `String[] args` | Array de Strings con los argumentos de línea de comandos |

```java
        System.out.println("¡Hola, Mundo!");
```

- **`System`**: Clase predefinida en `java.lang` (el único paquete con import automático).
- **`out`**: Campo `static final` de tipo `PrintStream`. Representa **stdout** (la consola).
- **`println(String)`**: Imprime el texto y añade un salto de línea. `print()` imprime sin salto.
- **`"¡Hola, Mundo!"`**: **String literal**. En Java, cadenas entre comillas dobles `"`. Caracteres individuales entre comillas simples `'a'`.
- **`;`**: El punto y coma es **obligatorio** al final de cada sentencia.

```java
    }    // Cierra el método main
}        // Cierra la clase HolaMundo
```

Las llaves delimitan bloques. La indentación (4 espacios por nivel) no afecta al compilador pero es fundamental para los humanos. No uses tabuladores — causan problemas entre editores.

### 1.5.3 Versión moderna con `var` (Java 10+)

```java
import java.util.ArrayList;
import java.util.List;

public class HolaMundoModerno {
    public static void main(String[] args) {
        var saludo = "¡Hola, Mundo!";           // String inferido
        var anio = 2026;                         // int inferido
        var pi = 3.141592653589793;              // double inferido
        var activo = true;                       // boolean inferido
        var nombres = new ArrayList<String>();   // ArrayList<String> inferido
        var lista = List.of("Ana", "Luis", "Carlos"); // List<String>

        System.out.println(saludo);
        System.out.println("Año: " + anio);
        System.out.println("Pi: " + pi);
        System.out.println("Nombres: " + nombres);
    }
}
```

**Reglas de `var`:**
- Solo para variables **locales** (dentro de métodos o bloques).
- Debe **inicializarse** en la misma línea (el compilador necesita el valor para inferir).
- No `var x = null;` — error de compilación (¿cuál sería el tipo?).
- El tipo se determina en compilación y es **fijo**. No puedes reasignar un tipo distinto.
- `var` no es palabra clave reservada — puedes tener una variable llamada `var` (aunque no deberías).

### 1.5.4 Text Blocks (Java 15+)

Los **text blocks** permiten escribir cadenas multilínea sin concatenar ni escapar. Se delimitan con **tres comillas dobles**:

```java
public class HolaTextBlock {
    public static void main(String[] args) {
        String poema = """
                ¡Hola, Mundo!
                Bienvenido a la programación en Java.

                Este es un text block:
                - Respeta los saltos de línea
                - No necesitas \\n
                - Las comillas "dobles" van sin escapar
                - La indentación común se elimina automáticamente
                """;

        System.out.println(poema);
    }
}
```

**Reglas:**
- Delimitado por `"""` de apertura y `"""` de cierre.
- La indentación común (determinada por la posición de cierre) se elimina.
- Usa `\` al final de línea para suprimir el salto de línea (útil para SQL).
- Ideal para HTML, JSON, SQL, XML, plantillas...

Ejemplos prácticos:

```java
// JSON con text block (antes era un infierno de escapes)
String json = """
        {
          "nombre": "Ana",
          "edad": 28,
          "activo": true
        }
        """;

// SQL con text block (ideal para consultas largas)
String query = """
        SELECT u.nombre, u.email, COUNT(p.id) AS total_pedidos
        FROM usuarios u
        LEFT JOIN pedidos p ON u.id = p.usuario_id
        WHERE u.activo = true
          AND u.fecha_registro > '2024-01-01'
        GROUP BY u.nombre, u.email
        ORDER BY total_pedidos DESC
        LIMIT 10
        """;
```

### 1.5.5 `System.out.printf()` — Salida formateada

```java
public class HolaFormateado {
    public static void main(String[] args) {
        String nombre = "María";
        int edad = 28;
        double altura = 1.72;
        double salario = 45000.50;

        System.out.printf("Nombre:     %s%n", nombre);
        System.out.printf("Edad:       %d años%n", edad);
        System.out.printf("Altura:     %.2f metros%n", altura);
        System.out.printf("Salario:    $%,.2f%n", salario);

        // Tabla alineada
        System.out.printf("%n| %-20s | %4s | %10s |%n", "NOMBRE", "EDAD", "SALARIO");
        System.out.printf("| %-20s | %4d | $%,9.2f |%n", "Ana García", 28, 45000.0);
        System.out.printf("| %-20s | %4d | $%,9.2f |%n", "Luis Martínez", 35, 72000.0);

        // Relleno con ceros
        System.out.printf("ID: %05d%n", 42);  // 00042
    }
}
```

| Especificador | Tipo | Ejemplo |
|---|---|---|
| `%s` | String | `Hola` |
| `%d` | Entero decimal | `42` |
| `%f` | Float/Double | `3.141593` |
| `%.2f` | Con 2 decimales | `3.14` |
| `%,.2f` | Con separadores | `1,234.56` |
| `%b` | Boolean | `true` |
| `%c` | Carácter | `A` |
| `%x`/`%X` | Hexadecimal | `ff`/`FF` |
| `%n` | Salto de línea | (portable) |
| `%%` | Literal `%` | `%` |

**Prefiere `%n` sobre `\n`**: usa el separador nativo de la plataforma.

### 1.5.6 Códigos de salida y `System.exit()`

```java
public class HolaSalida {
    public static void main(String[] args) {
        if (args.length == 0) {
            System.err.println("ERROR: Debes proporcionar tu nombre.");
            System.err.println("Uso: java HolaSalida <nombre>");
            System.exit(1);  // Salir con código de error
        }

        String nombre = args[0];
        if (nombre.length() < 2) {
            System.err.println("ERROR: El nombre debe tener al menos 2 caracteres.");
            System.exit(2);  // Diferente código para cada tipo de error
        }

        System.out.println("¡Hola, " + nombre + "!");
        // System.exit(0) es implícito si main termina normalmente
    }
}
```

En Linux/macOS:

```bash
java HolaSalida
echo $?    # → 1  (error)
java HolaSalida Andrés
echo $?    # → 0  (éxito)
```

En Windows:

```batch
java HolaSalida
echo %ERRORLEVEL%
```

- `0` = éxito. Cualquier otro número (1-255) = error.
- `System.exit()` fuerza la terminación inmediata. Ejecuta *shutdown hooks* antes de salir.
- `System.err` es el flujo de salida de error (stderr), separado de `System.out`.

### 1.5.7 Argumentos al `main`

```java
public class HolaArgumentos {
    public static void main(String[] args) {
        System.out.println("Número de argumentos: " + args.length);
        System.out.println("─── Lista de argumentos ───");
        for (int i = 0; i < args.length; i++) {
            System.out.printf("  args[%d] = \"%s\"%n", i, args[i]);
        }
    }
}
```

```bash
javac HolaArgumentos.java
java HolaArgumentos Ana 28 "San José, Costa Rica" true
```

Salida:

```
Número de argumentos: 4
─── Lista de argumentos ───
  args[0] = "Ana"
  args[1] = "28"
  args[2] = "San José, Costa Rica"
  args[3] = "true"
```

**Importante**: Los argumentos **siempre son Strings**. Si necesitas números:

```java
int edad = Integer.parseInt(args[1]);        // "28" → 28
double precio = Double.parseDouble(args[2]); // "19.99" → 19.99
```

Usa comillas en la terminal para argumentos con espacios.

---

## 1.6 Compilación y ejecución

### 1.6.1 ¿Qué hace realmente `javac`?

```bash
javac HolaMundo.java
```

Pipeline de compilación:

```
┌───────────────────────────────────────────────────────────────┐
│                FASES DE COMPILACIÓN DE javac                  │
│                                                               │
│  HolaMundo.java (texto Unicode)                               │
│      │                                                        │
│      ▼                                                        │
│  1. ANÁLISIS LÉXICO (Lexer)                                   │
│     Convierte caracteres en tokens:                           │
│     "public", "class", "HolaMundo", "{", "String"             │
│     Descarta comentarios y espacios.                          │
│      │                                                        │
│      ▼                                                        │
│  2. ANÁLISIS SINTÁCTICO (Parser)                              │
│     Construye un AST según la gramática de Java.              │
│     Verifica que la estructura es válida.                     │
│      │                                                        │
│      ▼                                                        │
│  3. ANÁLISIS SEMÁNTICO                                        │
│     Verifica tipos, referencias, ámbito y accesos.            │
│     "¿Existe String? ¿int compatible con boolean?"            │
│      │                                                        │
│      ▼                                                        │
│  4. GENERACIÓN DE BYTECODE                                     │
│     Traduce AST a instrucciones bytecode + constant pool.     │
│      │                                                        │
│      ▼                                                        │
│  HolaMundo.class (bytecode listo para la JVM)                 │
└───────────────────────────────────────────────────────────────┘
```

Si hay errores:

```
HolaMundo.java:5: error: ';' expected
        System.out.println("¡Hola, Mundo!")
                                          ^
1 error
```

El caret `^` señala exactamente dónde se detectó el problema. En Unix, "sin noticias son buenas noticias".

### 1.6.2 Inspeccionar bytecode con `javap`

```bash
javap -c HolaMundo          # Bytecode con instrucciones
javap -v HolaMundo          # Verbose: constant pool, atributos
javap -p HolaMundo          # Miembros privados
javap -c -v -p HolaMundo    # Todo junto
```

Ejemplo con `javap -c HolaMundo`:

```
Compiled from "HolaMundo.java"
public class HolaMundo {
  public HolaMundo();
    Code:
       0: aload_0
       1: invokespecial #1    // Method java/lang/Object."<init>":()V
       4: return

  public static void main(java.lang.String[]);
    Code:
       0: getstatic     #7    // Field java/lang/System.out:Ljava/io/PrintStream;
       3: ldc           #13   // String ¡Hola, Mundo!
       5: invokevirtual #15   // Method java/io/PrintStream.println:(Ljava/lang/String;)V
       8: getstatic     #7
      11: ldc           #21   // String Bienvenido a Java.
      13: invokevirtual #15
      16: return
}
```

Fíjate: el compilador generó un **constructor por defecto** automáticamente, aunque no escribimos ninguno. Si no defines constructores, Java añade uno público sin parámetros.

### 1.6.3 Crear un JAR ejecutable

Un **JAR** (Java ARchive) es un ZIP con tus `.class`, recursos y un `META-INF/MANIFEST.MF`. Es la unidad estándar de distribución.

```bash
# Crear JAR ejecutable
jar -cfe mi-app.jar HolaMundo *.class
#  -c  Crear nuevo JAR
#  -f  Nombre del archivo
#  -e  Clase principal (entry point)

# Ejecutar
java -jar mi-app.jar

# Inspeccionar contenido
jar -tf mi-app.jar
# META-INF/
# META-INF/MANIFEST.MF
# HolaMundo.class

# Extraer para inspeccionar
jar -xf mi-app.jar
cat META-INF/MANIFEST.MF
```

El MANIFEST.MF generado:

```
Manifest-Version: 1.0
Created-By: 21.0.1 (Eclipse Adoptium)
Main-Class: HolaMundo
```

`Main-Class` es lo que permite `java -jar`. Sin ella:

```bash
java -cp mi-app.jar HolaMundo
```

MANIFEST personalizado para proyectos reales:

```
Manifest-Version: 1.0
Main-Class: com.miempresa.MiAplicacion
Implementation-Title: Sistema de Pedidos
Implementation-Version: 2.1.0
Implementation-Vendor: Mi Empresa S.A.
Class-Path: lib/dependencia1.jar lib/dependencia2.jar
```

Empaquetar con MANIFEST personalizado:

```bash
jar -cfm mi-app.jar META-INF/MANIFEST.MF -C build/classes .
```

### 1.6.4 Compilar vs construir (build)

Una distinción crucial:

- **Compilar**: Transformar `.java` → `.class`. Es UNA fase. Comando: `javac`.
- **Construir (build)**: El proceso completo: compilar, ejecutar tests, empaquetar en JAR/WAR, copiar recursos, verificar dependencias, generar documentación... Comando: `mvn package` o `gradle build`.

En proyectos reales con 500+ clases y 50+ dependencias, llamar a `javac` manualmente es imposible. Maven y Gradle automatizan todo.

---

## 1.7 Estructura de un programa Java

### 1.7.1 Paquetes (packages)

Un **paquete** organiza clases en espacios de nombres jerárquicos. Se corresponde con la estructura de directorios en el sistema de archivos:

```
Estructura en disco:               Nombre completo de clase (FQCN):

src/
└── com/
    └── miapp/
        ├── modelo/                com.miapp.modelo.Usuario
        │   └── Usuario.java       com.miapp.modelo.Direccion
        ├── servicio/              com.miapp.servicio.UsuarioService
        │   └── UsuarioService.java
        └── utilidad/              com.miapp.utilidad.Validador
            └── Validador.java
```

**Convenciones de nomenclatura:**
- Todo en **minúsculas**. Nunca mayúsculas en nombres de paquete.
- Usa tu **dominio invertido**: `com.empresa.proyecto.modulo`.
- Sin dominio propio: `org.miproyecto` o `io.github.tuusuario`.
- Evita nombres genéricos: prefiere `validacion` sobre `util`.

**Beneficios:**
1. **Organización lógica**: Agrupa clases con funcionalidad relacionada.
2. **Evita colisiones**: `com.banco.Cliente` y `org.otro.Cliente` son clases distintas.
3. **Control de acceso**: El modificador *package-private* (sin modificador) limita visibilidad al mismo paquete.

### 1.7.2 Imports

La palabra clave `import` evita escribir el nombre completo:

```java
// SIN imports (horrible)
java.util.List<String> miLista = new java.util.ArrayList<>();
java.util.Map<String, Integer> miMapa = new java.util.HashMap<>();

// CON imports (limpio)
import java.util.List;
import java.util.Map;
import java.util.ArrayList;
import java.util.HashMap;

List<String> miLista = new ArrayList<>();
Map<String, Integer> miMapa = new HashMap<>();
```

Tipos de imports:

```java
import java.util.List;              // Específico (RECOMENDADO)
import java.util.*;                 // Wildcard (EVITAR en código serio)
import static java.lang.Math.PI;    // Import estático
import static java.lang.Math.*;     // Todos los miembros estáticos de Math
```

**`java.lang.*` se importa automáticamente** en cada archivo Java. No necesitas `import java.lang.String;` ni `import java.lang.System;`.

### 1.7.3 Módulos (Java 9+ — Project Jigsaw)

Antes de Java 9, el JDK era un monolito: `rt.jar` (~60 MB) con **todas** las clases. Aunque solo usaras `List`, cargabas `java.awt`, `javax.swing` y cientos más.

Project Jigsaw modularizó el JDK y permite modularizar tus aplicaciones. **¿Por qué importan los módulos?**

1. **Encapsulamiento fuerte**: Declaras qué paquetes exportas. El resto son internos e inaccesibles (incluso por reflexión).
2. **Runtimes ligeros**: `jlink` crea JVMs mínimas con solo los módulos necesarios. Un microservicio puede reducirse de 200MB a 30MB.
3. **Dependencias explícitas**: `requires` declara qué necesita tu módulo. La JVM verifica el grafo al arrancar.

Ejemplo de descriptor de módulo:

```java
// src/mi.modulo/module-info.java
module mi.modulo {
    requires java.sql;           // Necesito JDBC
    requires java.logging;       // Necesito logging

    exports com.miapp.api;       // API pública del módulo
    exports com.miapp.modelo;   // Modelos compartidos

    // Abro paquetes para reflexión (necesario para Hibernate/Spring)
    opens com.miapp.entidades to org.hibernate.orm.core;
}
```

Estructura con módulos:

```
src/
├── mi.modulo/
│   ├── module-info.java              ← descriptor del módulo
│   └── com/miapp/
│       ├── api/          (exportado: visible desde fuera)
│       ├── modelo/       (exportado)
│       └── interno/      (NO exportado: encapsulado)
```

**Nota para este libro**: La mayoría de aplicaciones empresariales aún no usan módulos explícitos (usan el classpath tradicional). Pero debes saber que existen, especialmente si trabajas con microservicios y `jlink`.

### 1.7.4 Classpath vs Module Path

| | Classpath (tradicional) | Module Path (Java 9+) |
|---|---|---|
| Búsqueda | Secuencial en JARs/directorios | Resuelve grafo de módulos declarados |
| Encapsulamiento | Débil (reflexión accede a todo) | Fuerte (solo exports) |
| Errores | `ClassNotFoundException` en runtime | Se detecta al arrancar |
| JAR Hell | Posible: clases duplicadas | Resuelto: el sistema lo detecta |
| Uso actual | Mayoritario | Creciente |

```bash
# Con classpath
java -cp lib/dep.jar:lib/otra.jar com.miapp.Main

# Con module path
java --module-path lib/ -m mi.modulo/com.miapp.Main
```

En la práctica, Maven y Gradle abstraen toda esta complejidad. Declaras dependencias y ellos construyen el classpath/module path correcto.

### 1.7.5 Organización de un proyecto real

Un proyecto Java profesional sigue esta estructura estándar (impuesta por Maven, adoptada por Gradle):

```
mi-proyecto/
│
├── pom.xml                     # Configuración Maven (o build.gradle)
├── README.md
├── .gitignore
│
├── src/
│   ├── main/
│   │   ├── java/               # CÓDIGO FUENTE PRINCIPAL
│   │   │   └── com/
│   │   │       └── empresa/
│   │   │           └── proyecto/
│   │   │               ├── MiAplicacion.java        (punto de entrada)
│   │   │               ├── controlador/             (capa HTTP/REST)
│   │   │               ├── servicio/                (lógica de negocio)
│   │   │               │   ├── UsuarioService.java  (interfaz)
│   │   │               │   └── UsuarioServiceImpl.java (implementación)
│   │   │               ├── repositorio/             (acceso a datos)
│   │   │               ├── modelo/                  (entidades)
│   │   │               ├── dto/                     (objetos transferencia)
│   │   │               ├── configuracion/           (configuración)
│   │   │               └── excepcion/               (excepciones propias)
│   │   │
│   │   └── resources/          # RECURSOS (no son código .java)
│   │       ├── application.properties
│   │       ├── application-dev.properties
│   │       ├── application-prod.properties
│   │       ├── db/migraciones/
│   │       ├── static/         # Recursos web estáticos (css, js, imágenes)
│   │       └── plantillas/     # Plantillas HTML, emails, etc.
│   │
│   └── test/
│       ├── java/               # TESTS
│       │   └── com/empresa/proyecto/
│       │       ├── servicio/
│       │       ├── controlador/
│       │       └── repositorio/
│       │
│       └── resources/
│           └── application-test.properties
│
└── target/                     # Build output (generado por Maven)
    ├── classes/                # .class compilados
    ├── mi-proyecto-1.0.0.jar   # JAR final
    └── test-classes/           # Tests compilados
```

**Principios de esta estructura:**
1. **Separación clara**: Código, recursos y tests en carpetas independientes.
2. **Estandarización universal**: Todos los proyectos Java del mundo la usan.
3. **Soporte nativo de herramientas**: Maven, Gradle, IntelliJ, Eclipse la reconocen sin configuración.
4. **Escalabilidad**: Funciona para 10 clases o 10,000.

### 1.7.6 Orden canónico de elementos en una clase

Por convención, los miembros de una clase siguen este orden:

```java
public class OrdenCanonico {

    // 1. CONSTANTES (public static final)
    public static final int MAXIMO_USUARIOS = 1000;
    public static final String VERSION = "2.1.0";

    // 2. ATRIBUTOS ESTÁTICOS
    private static int contadorInstancias = 0;
    private static final Logger log = Logger.getLogger(OrdenCanonico.class.getName());

    // 3. ATRIBUTOS DE INSTANCIA (campos)
    private String id;
    private String nombre;
    private LocalDateTime fechaCreacion;

    // 4. CONSTRUCTORES
    public OrdenCanonico() {
        this("sin-nombre");
    }

    public OrdenCanonico(String nombre) {
        this.id = UUID.randomUUID().toString();
        this.nombre = nombre;
        this.fechaCreacion = LocalDateTime.now();
        contadorInstancias++;
    }

    // 5. MÉTODOS PÚBLICOS (la API de la clase)
    public String getId() { return id; }
    public String getNombre() { return nombre; }
    public void actualizarNombre(String nuevoNombre) {
        validarNombre(nuevoNombre);
        this.nombre = nuevoNombre;
    }

    // 6. MÉTODOS PROTEGIDOS (para subclases)
    protected void antesDeGuardar() { }

    // 7. MÉTODOS PRIVADOS (implementación interna)
    private void validarNombre(String nombre) {
        if (nombre == null || nombre.isBlank()) {
            throw new IllegalArgumentException("El nombre no puede estar vacío");
        }
    }

    // 8. CLASES INTERNAS
    public enum Estado { ACTIVO, INACTIVO }
    private static class Builder { }
}
```

### 1.7.7 Convenciones de nomenclatura

| Elemento | Convención | Buen ejemplo | Mal ejemplo |
|---|---|---|---|
| Paquetes | `minúsculas` | `com.empresa.proyecto.modelo` | `com.Empresa.Proyecto` |
| Clases, Enums, Records | `PascalCase` | `UsuarioService`, `TipoDoc` | `usuario_service` |
| Métodos | `camelCase` (verbo) | `calcularTotal()`, `esValido()` | `CalcularTotal()` |
| Variables | `camelCase` (sustantivo) | `numeroEstudiantes` | `n`, `total1` |
| Constantes | `MAYÚSCULAS_GUIONES` | `MAX_CONEXIONES` | `maxConexiones` |
| Genéricos | `UnaLetraMayúscula` | `T`, `E`, `K`, `V` | `Tipo` |

**Reglas adicionales:**
- Booleanos: `isActivo`, `hasPermisos`, `canDelete`, `shouldRetry`. Las condiciones se leen como inglés.
- Métodos de acceso: `getNombre()` / `setNombre(String)`. Estándar JavaBeans reconocido por frameworks.
- Métodos que retornan: `findX()`, `createY()`, `calculateZ()`.
- Métodos de acción: `save()`, `delete()`, `process()`, `validate()`.
- Nombres descriptivos. El código se lee 10 veces más de lo que se escribe; optimiza para lectura.

---

## 1.8 Herramientas del ecosistema Java

### 1.8.1 Maven y Gradle: sistemas de construcción (build tools)

En proyectos reales con cientos de clases, decenas de dependencias y pipelines CI/CD, no llamas a `javac` manualmente. Usas una **build tool**.

#### Maven: el estándar empresarial

**Apache Maven** se basa en *Convention over Configuration*: sigue la estructura estándar y funciona sin casi configuración.

Archivo `pom.xml` realista:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0
         http://maven.apache.org/xsd/maven-4.0.0.xsd">

    <modelVersion>4.0.0</modelVersion>

    <!-- Coordenadas únicas del proyecto -->
    <groupId>com.miempresa</groupId>
    <artifactId>mi-aplicacion</artifactId>
    <version>1.0.0-SNAPSHOT</version>
    <packaging>jar</packaging>

    <properties>
        <java.version>21</java.version>
        <spring-boot.version>3.2.5</spring-boot.version>
    </properties>

    <dependencies>
        <!-- Spring Boot Starter Web (REST + Tomcat) -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
            <version>${spring-boot.version}</version>
        </dependency>

        <!-- H2 para desarrollo local -->
        <dependency>
            <groupId>com.h2database</groupId>
            <artifactId>h2</artifactId>
            <version>2.2.224</version>
            <scope>runtime</scope>
        </dependency>

        <!-- Testing -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <version>${spring-boot.version}</version>
            <scope>test</scope>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-compiler-plugin</artifactId>
                <version>3.13.0</version>
                <configuration>
                    <source>${java.version}</source>
                    <target>${java.version}</target>
                </configuration>
            </plugin>
        </plugins>
    </build>
</project>
```

**Ciclo de vida de Maven** (fases secuenciales):

```bash
mvn clean      # 1. Elimina target/
mvn validate   # 2. Valida el proyecto
mvn compile    # 3. Compila → target/classes
mvn test       # 4. Ejecuta tests unitarios
mvn package    # 5. Empaqueta en JAR/WAR
mvn verify     # 6. Tests de integración
mvn install    # 7. Instala en repo local (~/.m2/)
mvn deploy     # 8. Publica en repo remoto
```

Comandos esenciales:

```bash
mvn clean package -DskipTests    # Compilar y empaquetar rápido
mvn test                         # Solo ejecutar tests
mvn dependency:tree              # Ver árbol de dependencias (depurar conflictos)
mvn versions:display-dependency-updates  # Actualizaciones disponibles
```

#### Gradle: moderno y flexible

Gradle usa un DSL en Groovy o Kotlin. Es el build system oficial de Android.

```kotlin
// build.gradle.kts (Kotlin DSL — la variante moderna)
plugins {
    java
    id("org.springframework.boot") version "3.2.5"
}

group = "com.miempresa"
version = "1.0.0-SNAPSHOT"

java {
    sourceCompatibility = JavaVersion.VERSION_21
}

repositories {
    mavenCentral()
}

dependencies {
    implementation("org.springframework.boot:spring-boot-starter-web")
    testImplementation("org.junit.jupiter:junit-jupiter:5.10.1")
}

tasks.withType<Test> {
    useJUnitPlatform()
}
```

Comandos Gradle:

```bash
gradle build       # Compilar + test + empaquetar
gradle test        # Solo tests
gradle bootRun     # Ejecutar app Spring Boot
gradle dependencies # Árbol de dependencias
```

#### ¿Maven o Gradle?

| | Maven | Gradle |
|---|---|---|
| Configuración | XML (verboso) | Groovy/Kotlin DSL (conciso) |
| Curva aprendizaje | Baja | Media |
| Velocidad | Buena | Excelente (caché incremental) |
| Flexibilidad | Baja (convenciones rígidas) | Alta |
| Industria | Mayoritario | Creciente |
| Aprender primero | **Maven** | Después |

**Recomendación**: Aprende Maven primero. Es más fácil, predecible y ubicuo en empresas.

### 1.8.2 IDEs (Entornos de Desarrollo Integrados)

#### IntelliJ IDEA: el estándar de oro

La **Community Edition** es gratuita y cubre el 90% de necesidades. La Ultimate (pago) añade soporte para Spring, JPA, DB.

**Por qué IntelliJ:**
- Autocompletado inteligente que predice lo que quieres escribir
- Refactorización poderosa y segura (renombrar, extraer método, mover clase)
- Análisis de código en tiempo real (te avisa de bugs mientras escribes)
- Integración nativa: Maven, Gradle, Git, Docker, DB
- Depuración visual con breakpoints condicionales

**Atajos esenciales de IntelliJ:**

| Atajo | Acción |
|---|---|
| `Ctrl+Shift+A` | Buscar cualquier acción |
| `Ctrl+N` | Buscar clase por nombre |
| `Ctrl+Shift+N` | Buscar archivo |
| `Alt+Enter` | **EL MÁS IMPORTANTE**: acción sugerida |
| `Ctrl+Alt+L` | Formatear código |
| `Shift+F6` | Renombrar (refactorizar) |
| `Shift+F9` | Ejecutar en modo debug |
| `F8` | Step Over (debug) |
| `F7` | Step Into (debug) |

#### Comparativa de IDEs

| | IntelliJ IDEA | Eclipse | VS Code |
|---|---|---|---|
| Precio | Community gratis | Gratuito | Gratuito |
| Peso | Medio | Pesado | Ligero |
| Refactorización | Excelente | Muy buena | Buena (básica) |
| Soporte Spring | Excelente (Ultimate) | Buena | Aceptable |
| Ideal para | Todo uso | Corporativo | Políglotas |

**Recomendación**: Instala IntelliJ IDEA Community Edition. Es gratis, es el más usado y te acompañará desde tu primer HolaMundo hasta sistemas enterprise.

### 1.8.3 JShell: Java interactivo (REPL)

Desde Java 9, el JDK incluye **JShell**, un REPL que ejecuta código Java sin crear clases, métodos ni compilar:

```bash
jshell
```

```
|  Welcome to JShell -- Version 21.0.1

jshell> 2 + 2
$1 ==> 4

jshell> var saludo = "Hola desde JShell"
saludo ==> "Hola desde JShell"

jshell> saludo.toUpperCase()
$3 ==> "HOLA DESDE JSHELL"

jshell> List.of("Ana", "Luis", "Carlos")
   ...>     .stream()
   ...>     .filter(n -> n.startsWith("A"))
   ...>     .toList()
$5 ==> [Ana]

jshell> /vars              # Ver variables definidas
jshell> /methods           # Ver métodos definidos
jshell> /list              # Todo lo ejecutado
jshell> /edit <id>         # Editar una declaración previa
jshell> /save sesion.jsh   # Guardar sesión en archivo
jshell> /open sesion.jsh   # Cargar sesión desde archivo
jshell> /reset             # Reiniciar sesión
jshell> /exit              # Salir
```

JShell es perfecto para probar APIs, verificar cómo funciona un método, o experimentar con conceptos nuevos sin la fricción de crear un proyecto completo.

### 1.8.4 Depuración básica

Encontrar y corregir bugs es el 40% del trabajo de un programador. Aprender a depurar es la decisión más rentable de tu carrera.

#### Depuración con IntelliJ

1. **Pon un breakpoint**: Clic en el margen izquierdo del editor (punto rojo).
2. **Ejecuta en modo debug**: Botón verde con bicho, o `Shift+F9`.
3. Cuando el programa pause en el breakpoint:
   - **Inspecciona variables** pasando el mouse sobre ellas
   - **F8 (Step Over)**: Ejecutar la línea actual y pasar a la siguiente
   - **F7 (Step Into)**: Entrar dentro del método que se invoca
   - **Shift+F8 (Step Out)**: Salir del método actual
   - **Alt+F8 (Evaluate Expression)**: Ejecutar código arbitrario en ese punto

```java
public class Calculadora {
    public static double dividir(int a, int b) {
        return (double) a / b;
    }

    public static void main(String[] args) {
        int x = 10;
        int y = 0;
        // ← Pon un breakpoint aquí
        double resultado = dividir(x, y);  // ¡División por cero!
        System.out.println("Resultado: " + resultado);
    }
}
```

#### `jdb`: depurador de línea de comandos

Para servidores remotos sin GUI:

```bash
# Arrancar JVM en modo debug (puerto 5005)
java -agentlib:jdwp=transport=dt_socket,server=y,suspend=y,address=5005 MiApp

# En otra terminal, conectar jdb
jdb -attach 5005

# Comandos útiles dentro de jdb:
> stop at MiApp:10     # Breakpoint en línea 10
> run                  # Continuar ejecución
> print variable       # Imprimir valor
> step                 # Siguiente línea
> step up              # Salir del método actual
> locals               # Variables locales
> where                # Stack trace
> exit
```

En la práctica, usarás el depurador del IDE el 95% del tiempo, pero `jdb` existe para emergencias en producción.

### 1.8.5 Logging con `java.util.logging`

`System.out.println()` es para aprender. En producción necesitas **logging**: mensajes con niveles de severidad, timestamps y destinos configurables.

```java
import java.util.logging.Level;
import java.util.logging.Logger;

public class HolaLogging {
    private static final Logger log =
        Logger.getLogger(HolaLogging.class.getName());

    public static void main(String[] args) {
        log.info("Iniciando la aplicación...");

        try {
            int resultado = dividir(10, 0);
            log.info("Resultado: " + resultado);
        } catch (ArithmeticException e) {
            log.log(Level.SEVERE, "Error al dividir", e);
        }

        log.fine("Mensaje de depuración detallada");
        log.warning("Esto es un aviso");
        log.info("Aplicación finalizada");
    }

    private static int dividir(int a, int b) {
        log.fine(() -> "Dividiendo " + a + " entre " + b);
        return a / b;
    }
}
```

**Niveles de logging (de menor a mayor severidad):**

| Nivel | Uso típico |
|---|---|
| `FINEST` | Traza ultra-detallada (valores en bucles) |
| `FINER` | Traza detallada |
| `FINE` | Información de depuración |
| `CONFIG` | Mensajes de configuración |
| `INFO` | Información general (arranque, hitos) |
| `WARNING` | Avisos (algo inesperado pero no crítico) |
| `SEVERE` | Errores graves que requieren atención |

Por defecto solo se muestra `INFO` y superior. En proyectos reales usarás SLF4J + Logback (lo veremos en capítulos avanzados).

---

## 1.9 El modelo mental del programador Java

Esta sección es, posiblemente, la más importante del capítulo. No es sintaxis — es **cómo pensar**. Cuando entiendes lo que ocurre debajo del capó, los comportamientos "misteriosos" de Java desaparecen.

### 1.9.1 Stack vs Heap: dónde viven tus datos

La JVM divide la memoria en dos regiones fundamentales:

```
┌───────────────────────────────────────────────────────────────────────────┐
│                    MEMORIA DE UN PROGRAMA JAVA EN EJECUCIÓN                │
│                                                                           │
│   ┌───────────────────────────────┐   ┌────────────────────────────────┐  │
│   │           STACK               │   │             HEAP               │  │
│   │        (La Pila)              │   │          (El Montón)           │  │
│   │                               │   │                                │  │
│   │  ┌──────────────────┐         │   │  ┌─────────────────────────┐   │  │
│   │  │ Frame de main    │         │   │  │ Objeto String "Hola"    │   │  │
│   │  │  args → [ ]      │         │   │  └─────────────────────────┘   │  │
│   │  │  numero = 42     │         │   │                                │  │
│   │  │  texto ──────────┼─────┐   │   │  ┌─────────────────────────┐   │  │
│   │  └──────────────────┘     │   │   │  │ new Persona("Ana")      │   │  │
│   │                           │   │   │  │  nombre = "Ana"         │   │  │
│   │  ┌──────────────────┐     └───┼──►│  │  edad = 28              │   │  │
│   │  │ Frame de método  │         │   │  └─────────────────────────┘   │  │
│   │  │  a = 10          │         │   │                                │  │
│   │  │  b = 20          │         │   │  ┌─────────────────────────┐   │  │
│   │  │  suma = 30       │         │   │  │ int[] {1, 2, 3, 4, 5}   │   │  │
│   │  └──────────────────┘         │   │  └─────────────────────────┘   │  │
│   │                               │   │                                │  │
│   │  Cada hilo tiene su           │   │  Compartido por todos          │  │
│   │  propio STACK                 │   │  los hilos                     │  │
│   └───────────────────────────────┘   └────────────────────────────────┘  │
│                                                                           │
│   ┌──────────────────────────────────────────────────────────────────┐    │
│   │                           METASPACE                               │    │
│   │  class Persona { nombre, edad, getNombre()... }                   │    │
│   │  class String { value[], hash, length()... }                      │    │
│   │  static int contador = 0;                                         │    │
│   │  Metadatos de clases, constant pool, variables estáticas          │    │
│   └──────────────────────────────────────────────────────────────────┘    │
└───────────────────────────────────────────────────────────────────────────┘
```

#### El Stack (Pila)

- **Qué almacena**: Variables locales primitivas (`int`, `double`, `boolean`) y **referencias** a objetos (no los objetos en sí).
- **Organización**: Pila LIFO. Cada método crea un *frame* al entrar; al salir, el frame se destruye y sus variables desaparecen.
- **Velocidad**: Muy rápido (asignación/liberación LIFO).
- **Tamaño**: Limitado. Recursión infinita produce `StackOverflowError`.
- **Visibilidad**: Privado por hilo. Cada hilo tiene su propio stack.

#### El Heap (Montón)

- **Qué almacena**: **Todos** los objetos creados con `new`. Strings, arrays, instancias de clases, colecciones...
- **Organización**: Sin orden fijo. El GC gestiona fragmentación y compactación.
- **Velocidad**: Más lento que el stack (requiere desreferenciar punteros).
- **Tamaño**: Grande (configurable con `-Xmx`). Puede ser de gigabytes.
- **Visibilidad**: Compartido entre todos los hilos.
- **Gestión**: Automática por el Garbage Collector.

#### Ejemplo visual

```java
public static void main(String[] args) {
    int numero = 42;                   // Stack: valor 42 directamente
    String texto = "Hola";             // Stack: referencia → Heap: "Hola"
    Persona p = new Persona("Ana");    // Stack: referencia → Heap: Persona
}
```

```
STACK                            HEAP
┌─────────────────┐              ┌──────────────────┐
│ Frame de main    │              │ String "Hola"    │
│  numero = 42     │              └──────────────────┘
│  texto ──────────┼──────────────► value = ['H','o','l','a']
│  p ──────────────┼────┐         ┌──────────────────┐
└─────────────────┘    │         │ Persona          │
                       └─────────► nombre ──────────► "Ana"
                                 │ edad = 0        │
                                 └──────────────────┘
```

**Regla fundamental**: Variables de tipos primitivos contienen el valor directamente. Variables de tipos de referencia contienen una "flecha" que apunta al objeto real en el heap.

### 1.9.2 Cómo se ejecuta realmente un programa Java

Sigamos la vida completa de `HolaMundo.java`:

```
FASE 1: ESCRIBIR
  Creas HolaMundo.java en un editor. Es solo texto Unicode.
  Estado: código fuente (.java)

FASE 2: COMPILAR
  $ javac HolaMundo.java
  javac → Análisis léxico → Análisis sintáctico (AST)
       → Análisis semántico → Generación de bytecode
  Resultado: HolaMundo.class (bytecode + constant pool + metadatos)
  Estado: bytecode (.class)

FASE 3: EMPAQUETAR (para distribución)
  $ jar -cfe mi-app.jar HolaMundo *.class
  Crea un ZIP con:
  ├── META-INF/MANIFEST.MF  (Main-Class: HolaMundo)
  └── HolaMundo.class
  Estado: JAR ejecutable

FASE 4: EJECUTAR
  $ java -jar mi-app.jar
  1. La JVM arranca como proceso del SO
  2. Bootstrap ClassLoader carga java.lang.*, java.util.*, ...
  3. Platform y Application ClassLoaders se inicializan
  4. Application CL busca HolaMundo.class en el classpath
  5. Bytecode Verifier inspecciona la seguridad del bytecode
  6. JVM busca: public static void main(String[] args)
  7. El intérprete ejecuta bytecode de main instrucción por instrucción
  8. El profiler cuenta invocaciones (preparando JIT)
  9. Métodos calientes → JIT compila a código máquina nativo
  10. System.out.println() → stdout → terminal
  Resultado: "¡Hola, Mundo!" en pantalla. Exit code: 0.

FASE 5: DEPURAR (cuando algo falla)
  1. Leer el stack trace (la JVM dice exactamente qué línea falló)
  2. Poner breakpoint cerca del fallo en el IDE
  3. Ejecutar en modo debug
  4. Inspeccionar variables, avanzar paso a paso
  5. Identificar la causa raíz
  6. Corregir el código
  7. Volver a FASE 2
```

### 1.9.3 El ciclo de desarrollo profesional

Tu día a día como programador Java:

```
┌─────────────────────────────────────────────────────────────────┐
│              CICLO DE DESARROLLO JAVA PROFESIONAL                │
│                                                                 │
│   ┌──────────┐     ┌──────────┐     ┌──────────┐               │
│   │ Escribir │────►│ Compilar │────►│ Ejecutar │               │
│   │ código   │     │ (mvn      │     │ tests    │               │
│   │          │     │  compile) │     │ (mvn test)│              │
│   └──────────┘     └──────────┘     └────┬─────┘               │
│        ▲                                 │                      │
│        │                         ┌───────▼────────┐             │
│        │                         │ ¿Tests pasan?  │             │
│        │                         └───┬────────┬───┘             │
│        │                        NO   │        │  SÍ             │
│        │                             ▼        ▼                 │
│        │                     ┌──────────┐  ┌──────────┐         │
│        │                     │ Depurar  │  │ Empaquetar│         │
│        │                     │ (debug)  │  │ (mvn      │         │
│        │                     │          │  │  package) │         │
│        │                     └────┬─────┘  └────┬─────┘         │
│        │                          │             │               │
│        └──────────────────────────┘             ▼               │
│                                       ┌──────────────────┐      │
│                                       │ Desplegar /       │      │
│                                       │ Ejecutar en prod  │      │
│                                       └──────────────────┘      │
└─────────────────────────────────────────────────────────────────┘
```

---

## 1.10 Resumen del capítulo

- **¿Qué es Java?** Mucho más que un lenguaje: un ecosistema que incluye JVM, herramientas de construcción, IDEs, bibliotecas estándar y lenguajes hermanos (Kotlin, Scala, Groovy). Java domina el backend empresarial, Android, Big Data y trading financiero gracias a su combinación única de portabilidad, rendimiento y madurez.

- **Historia**: Nació en 1995 del Green Team de Sun. Venció a C++ eliminando punteros y gestión manual de memoria. Perdió contra JavaScript en el navegador, pero conquistó los servidores. Bajo Oracle, sigue evolucionando con releases cada 6 meses y versiones LTS cada 2 años.

- **JVM, JRE, JDK**: La JVM ejecuta bytecode. ClassLoaders jerárquicos (Bootstrap → Platform → Application) cargan clases. El bytecode se interpreta inicialmente; el JIT (C1/C2, Tiered Compilation) compila los *hot spots* a código nativo. El GC (Serial, Parallel, G1, ZGC, Shenandoah) gestiona la memoria automáticamente.

- **Instalación**: Eclipse Temurin (Adoptium). SDKMAN! es la mejor herramienta para gestionar múltiples versiones en Linux/macOS. `JAVA_HOME` y `PATH` deben configurarse correctamente.

- **Primer programa**: `public class HolaMundo` + `main` + `System.out.println()`. Versiones modernas: `var` (Java 10), text blocks (Java 15), `printf()`, argumentos de línea de comandos, códigos de salida con `System.exit()`.

- **Compilación**: `javac` convierte `.java` en `.class` (bytecode). `javap -c` permite inspeccionar el bytecode generado. `jar -cfe` crea JARs ejecutables con MANIFEST.MF. Compilar no es lo mismo que construir (build): `mvn package`.

- **Estructura**: Paquetes organizan el código y evitan colisiones. Los módulos (Java 9+) añaden encapsulamiento fuerte. La estructura de proyecto estándar (`src/main/java`, `src/main/resources`, `src/test/java`) es universal gracias a Maven.

- **Herramientas**: Maven y Gradle automatizan el ciclo compilar-test-empaquetar. IntelliJ IDEA es el IDE estándar. JShell permite experimentar interactivamente. El depurador (IDE o `jdb`) es tu mejor amigo. El logging profesional reemplaza a `System.out.println()`.

- **Modelo mental**: Stack (variables primitivas, referencias) vs Heap (objetos, arrays). El ciclo de desarrollo profesional: escribir → compilar → ejecutar tests → depurar → empaquetar → desplegar.

En el próximo capítulo, nos sumergiremos en los **fundamentos del lenguaje**: tipos de datos, variables, operadores y estructuras de control. Pero ya tienes lo más importante: el mapa completo del territorio Java. Ahora solo queda explorarlo.

---

## Ejercicios del Capítulo 1

### Ejercicio 1: Hola Mundo personalizado

Escribe un programa `HolaPersonal.java` que imprima la siguiente información en líneas separadas usando tanto `println` como `print`:
- Tu nombre completo
- Tu ciudad y país
- Tu profesión actual o la que aspiras tener
- Un dato curioso sobre ti

**Objetivo**: Practicar la estructura básica de un programa, `main` y `System.out`.

---

### Ejercicio 2: Argumentos de línea de comandos

Escribe un programa `Presentacion.java` que reciba exactamente 3 argumentos: nombre, edad y profesión. Debe validar que se pasaron los 3 argumentos (si no, mostrar ayuda y salir con código de error). Si los argumentos son correctos, debe imprimir:

`"Me llamo [nombre], tengo [edad] años y soy [profesión]."`

Ejemplo de ejecución:

```bash
$ java Presentacion "Ana García" 28 "Ingeniera de Software"
Me llamo Ana García, tengo 28 años y soy Ingeniera de Software.

$ java Presentacion
Error: debes proporcionar 3 argumentos.
Uso: java Presentacion <nombre> <edad> <profesion>
```

**Objetivo**: Trabajar con `args[]`, `System.exit()` y `System.err`.

---

### Ejercicio 3: Exploración de bytecode

Escribe una clase `Operaciones.java` con tres métodos estáticos: `sumar(int a, int b)`, `multiplicar(int a, int b)` y `esPar(int n)`. Compílala y usa `javap -c Operaciones` para inspeccionar el bytecode generado.

Responde:
- ¿Cuántas instrucciones bytecode tiene cada método?
- ¿Qué instrucción se usa para la suma? ¿Y para la multiplicación?
- ¿Cómo determina el bytecode si un número es par? (pista: observa instrucciones como `irem`, `iand` o comparación con cero)
- Ejecuta `javap -v Operaciones` y localiza el constant pool. ¿Qué constantes aparecen?

**Objetivo**: Familiarizarte con `javap` y el concepto de bytecode.

---

### Ejercicio 4: Gestión de versiones con SDKMAN!

Si estás en Linux, macOS o WSL:
1. Instala SDKMAN! siguiendo las instrucciones de la sección 1.4.2
2. Instala al menos 2 versiones LTS de Java (por ejemplo: Java 17 y Java 21)
3. Cambia entre ellas con `sdk use` y verifica con `java -version`
4. Responde: ¿Qué diferencias ves entre las salidas de `java -version`? ¿Cambia el GC por defecto? ¿Cambia el modo de la VM?

**Objetivo**: Aprender a gestionar múltiples JDKs, una habilidad esencial para el trabajo profesional.

---

### Ejercicio 5: Crear un JAR ejecutable

Crea un programa `Saludo.jar` que:
1. Tenga una clase `Saludo` con método `main` que imprima un saludo y todos los argumentos recibidos
2. Compílalo y empaquétalo en un JAR ejecutable con el comando `jar -cfe`
3. Verifica que el archivo `META-INF/MANIFEST.MF` contiene `Main-Class: Saludo`
4. Ejecútalo con `java -jar Saludo.jar argumento1 argumento2`
5. (Extra) Crea un MANIFEST.MF personalizado que incluya `Implementation-Version: 1.0.0` y vuelve a empaquetar

**Objetivo**: Entender JARs, MANIFEST.MF y distribución de aplicaciones Java.

---

### Ejercicio 6: JShell interactivo

Abre JShell y completa las siguientes tareas sin crear archivos `.java`:

```java
// a) Define una variable con tu nombre
// b) Convierte tu nombre a mayúsculas con String.toUpperCase()
// c) Crea una lista de números: List<Integer> nums = List.of(1, 2, 3, 4, 5)
// d) Calcula la suma de los números: nums.stream().mapToInt(Integer::intValue).sum()
// e) Define un método: String saludar(String nombre) { return "Hola, " + nombre; }
// f) Invoca saludar() con tu nombre
// g) Guarda la sesión en un archivo con /save
// h) Sal de JShell, vuelve a entrar y carga la sesión con /open
// i) Usa /list para ver todo lo que hiciste
```

**Objetivo**: Experimentar con Java sin compilar, y aprender a usar JShell para prototipado rápido.

---

### Ejercicio 7: Organización de proyecto completo

Crea manualmente (sin Maven ni Gradle) la estructura de directorios para un proyecto llamado `BibliotecaVirtual` que contenga:

```
src/
├── main/
│   └── java/
│       └── com/
│           └── tusiniciales/
│               └── biblioteca/
│                   ├── Main.java
│                   ├── modelo/
│                   │   ├── Libro.java
│                   │   └── Autor.java
│                   └── servicio/
│                       └── BibliotecaService.java
└── test/
    └── java/
        └── com/
            └── tusiniciales/
                └── biblioteca/
                    └── servicio/
                        └── BibliotecaServiceTest.java
```

Implementa:
- `Libro`: atributos `titulo`, `autor`, `isbn`, `añoPublicacion` (usa `record` si estás en Java 17+)
- `Autor`: atributos `nombre`, `nacionalidad`
- `BibliotecaService`: métodos `agregarLibro(Libro)`, `buscarPorTitulo(String)`, `listarTodos()` que devuelve una lista
- `Main`: crea un servicio, agrega al menos 3 libros, busca uno por título e imprime el catálogo completo

Compila y ejecuta todo desde la línea de comandos usando classpath manual. Pista: necesitarás usar la opción `-d` de javac para compilar a un directorio de salida y luego usar `-cp` con java.

**Objetivo**: Interiorizar la estructura estándar de proyectos y practicar paquetes, imports y classpath manual.

---

### Ejercicio 8: Análisis del GC en vivo

1. Escribe un programa `GeneradorBasura.java` que cree 10 millones de Strings en un bucle y permita observar el comportamiento de la memoria:

```java
public class GeneradorBasura {
    public static void main(String[] args) throws Exception {
        System.out.println("PID: " + ProcessHandle.current().pid());
        System.out.println("Presiona Enter para empezar...");
        System.in.read();

        for (int i = 0; i < 10_000_000; i++) {
            String s = new String("Objeto número " + i);
            if (i % 1_000_000 == 0) {
                System.out.printf("Creados %,d objetos...%n", i);
                Thread.sleep(1000); // Pausa para observar el GC
            }
        }
        System.out.println("Terminado. Presiona Enter para salir.");
        System.in.read();
    }
}
```

2. Ejecútalo con una JVM configurada para mostrar actividad del GC:

```bash
javac GeneradorBasura.java
java -Xms64m -Xmx256m -Xlog:gc*:file=gc.log:time,level,tags GeneradorBasura
```

3. En otra terminal, mientras el programa corre:
   - Encuentra el PID con `jps -l`
   - Monitoriza el GC con: `jstat -gc <PID> 1000`
   - También prueba: `jstat -gcutil <PID> 1000`

4. Responde a estas preguntas observando la salida de `jstat`:
   - ¿Cuántos Young GC (YGC) ocurrieron durante los 10 millones de iteraciones?
   - ¿Ocurrió algún Full GC (FGC > 0)? ¿Por qué sí o por qué no?
   - ¿Cómo varía el uso de Eden (EU) durante la ejecución? ¿Sube y baja rápidamente (patrón de sierra)?
   - ¿Cuándo comienzan a promocionar objetos a Old Generation? (Observa OU > 0)
   - Si aumentas `-Xmx` a 512m, ¿cambia el número de GC?

5. (Extra) Repite el experimento con diferentes GCs:
   - `-XX:+UseSerialGC`
   - `-XX:+UseParallelGC`
   - `-XX:+UseG1GC`
   ¿Qué diferencias observas en la frecuencia y duración de las pausas?

**Objetivo**: Entender el Garbage Collector en acción, usar `jstat` para monitorización, y experimentar con diferentes algoritmos de GC.
