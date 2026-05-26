# Capítulo 12: Testing Profesional con JUnit y Mockito

> *"El testing no es una fase del proyecto. Es una disciplina de desarrollo que, bien aplicada, produce código más limpio, más mantenible y con menos defectos."* — Robert C. Martin

> *"Escribe pruebas hasta que el miedo se transforme en aburrimiento."* — Kent Beck

---

Cuando un programador novato escribe código, su instinto le dice: *"esto funciona, lo probé manualmente con tres casos"*. Cuando un programador profesional escribe código, su formación le dice: *"si no tiene pruebas automatizadas, no está terminado"*. El testing automatizado no es un complemento opcional ni una tarea que se delega al final del proyecto: es una práctica fundamental que define la calidad, la mantenibilidad y la velocidad de entrega de un equipo de desarrollo.

En este capítulo nos sumergiremos en el ecosistema de testing de Java moderno, abarcando desde los conceptos fundamentales hasta técnicas avanzadas como mutation testing, property-based testing y contract testing. No asumimos experiencia previa con frameworks de testing, pero sí un dominio sólido de Java y sus APIs fundamentales.

Empecemos con una verdad incómoda: **el código sin tests es código legacy desde el momento en que se escribe**. Porque código legacy no es código antiguo, sino código que no podemos modificar con confianza porque no sabemos qué romperá. Las pruebas automatizadas son la red de seguridad que nos permite evolucionar un sistema sin miedo.

---

## 12.1 La Pirámide del Testing

Imagina que eres un chef en un restaurante de tres estrellas Michelin. Antes de servir un plato, realizas múltiples verificaciones: pruebas cada ingrediente por separado (¿la lechuga está fresca?), pruebas combinaciones de ingredientes durante la cocción (¿la salsa liga bien con la pasta?) y, finalmente, pruebas el plato completo antes de enviarlo al comedor (¿el conjunto es armonioso?). Esta es exactamente la lógica detrás de la pirámide del testing.

### 12.1.1 Los Tres Niveles

La pirámide del testing, popularizada por Mike Cohn en su libro *Succeeding with Agile*, describe tres niveles de pruebas con diferentes características:

```
            /  E2E  \          ▲  Pocas pruebas
           /---------\         │  Lentas (segundos/minutos)
          / Integración \      │  Frágiles (dependen de entorno)
         /---------------\     │  Alto costo de mantenimiento
        /   Unitarias      \   ▼  Muchas pruebas
       /---------------------\  Rápidas (milisegundos)
      ------------------------- Estables (aisladas)
      Muchas │ Poco costo │ Alta confianza individual
```

> [!NOTE] 🍳 **Analogía del Restaurante:** Así como un chef no espera a que el plato esté terminado para probar la sopa (prueba unitaria), ni prueba el plato completo sin haber verificado que el horno funciona (prueba de integración), en software combinamos múltiples niveles de verificación. **En resumen:** cada nivel de la pirámide detecta tipos distintos de defectos, y un equilibrio adecuado entre ellos produce la suite de pruebas más eficiente.

#### Pruebas Unitarias (Base de la pirámide)

Verifican el comportamiento de una unidad aislada de código — típicamente un método o una clase — sin dependencias externas.

| Característica | Descripción |
|---|---|
| **Velocidad** | Milisegundos (cientos o miles por segundo) |
| **Aislamiento** | Sin red, sin BD, sin sistema de archivos |
| **Cobertura** | Validan lógica de negocio, casos límite |
| **Mantenibilidad** | Refactorizables, bajo acoplamiento |
| **Cantidad esperada** | Cientos o miles por proyecto |

```java
@Test
@DisplayName("Debería calcular el total con impuesto incluido")
void deberiaCalcularTotalConImpuesto() {
    CalculadoraFactura calculadora = new CalculadoraFactura(new TasasImpositivasFixtures());
    BigDecimal total = calculadora.calcularTotal(new LineaFactura("Teclado", new BigDecimal("79.99"), 2));
    assertThat(total).isEqualByComparingTo(new BigDecimal("175.98"));
}
```

#### Pruebas de Integración (Centro de la pirámide)

Verifican la interacción entre múltiples componentes reales: tu código con una base de datos, con una API REST, con un broker de mensajes.

| Característica | Descripción |
|---|---|
| **Velocidad** | Segundos (requieren inicializar recursos) |
| **Dependencias** | Base de datos, contenedores Docker, APIs |
| **Cobertura** | Validan contratos entre componentes |
| **Mantenibilidad** | Más frágiles, requieren gestión de infraestructura |
| **Cantidad esperada** | Decenas o pocos cientos |

```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@Testcontainers
class PedidoControllerIntegrationTest {

    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16-alpine");

    @DynamicPropertySource
    static void configurarPropiedades(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
    }

    @Test
    @DisplayName("Debería crear un pedido y persistirlo en la base de datos real")
    void deberiaCrearPedidoYPersistirlo() {
        // Arrange: preparamos el request
        // Act: invocamos el endpoint REST
        // Assert: verificamos que el pedido existe en la BD
    }
}
```

#### Pruebas End-to-End (Cima de la pirámide)

Verifican el sistema completo desde la perspectiva del usuario final. Son las más lentas, las más costosas y las más frágiles. Deberían ser las menos numerosas.

| Característica | Descripción |
|---|---|
| **Velocidad** | Minutos (despliegan toda la aplicación) |
| **Entorno** | Réplica de producción (staging) |
| **Cobertura** | Validan flujos críticos de negocio |
| **Mantenibilidad** | Muy frágiles, dependen de todo el stack |
| **Cantidad esperada** | Pocas (decenas, máximo) |

> [!NOTE] ⚠️ **El Anti-Patrón del Cono de Helado:** Muchos equipos caen en la trampa de tener una pirámide invertida: muchas pruebas E2E, moderadas de integración y pocas unitarias. Esto produce suites lentas (30-60 minutos), frágiles (falsos positivos constantes) y costosas de mantener. **En resumen:** invierte en pruebas unitarias primero, integración después, y E2E solo para flujos críticos.

### 12.1.2 El Trofeo del Testing (Testing Trophy)

Kent C. Dodds propuso una evolución de la pirámide llamada *Testing Trophy* (Trofeo del Testing), que enfatiza las pruebas de integración como el centro de la estrategia:

```
       /    E2E     \
      /--------------\
     /   Integración   \    ← ÉNFASIS PRINCIPAL
    /--------------------\
   /   Unitarias (estáticas) \
  /----------------------------\
```

La diferencia clave es que Dodds argumenta que las pruebas de integración — aquellas que verifican la interacción entre componentes sin mockear excesivamente — ofrecen el mejor retorno de inversión: son razonablemente rápidas, detectan problemas reales de integración y no sufren del falso positivismo de los mocks excesivos.

> [!NOTE] 🏆 **Trofeo vs Pirámide:** La pirámide prioriza las unitarias por velocidad y aislamiento; el trofeo prioriza las de integración porque detectan más bugs reales. **En resumen:** no son opuestos sino complementarios: usa la pirámide como guía general y el trofeo como recordatorio de que mockearlo todo no garantiza que el sistema funcione.

### 12.1.3 Principios FIRST

Toda prueba automatizada debería cumplir cinco propiedades, resumidas en el acrónimo **FIRST**:

| Principio | Significado | Por qué importa |
|---|---|---|
| **F**ast (Rápida) | Ejecutarse en milisegundos | Si tardan segundos, dejarás de ejecutarlas frecuentemente |
| **I**ndependent (Independiente) | No depender de otras pruebas ni de orden de ejecución | Una prueba falla por su propio defecto, no por estado residual |
| **R**epeatable (Repetible) | Mismo resultado en cualquier entorno | Sin depender del reloj, la red o el sistema de archivos |
| **S**elf-validating (Auto-validable) | Resultado booleano: pasa o falla | Sin necesidad de inspección manual del output |
| **T**imely (Oportuna) | Escrita justo antes o junto al código de producción | Probar código legacy es mucho más costoso |

```java
// MAL: prueba no repetible — depende del reloj del sistema
@Test
void deberiaCalcularEdad() {
    Persona persona = new Persona("María", LocalDate.of(1990, 5, 15));
    int edadEsperada = LocalDate.now().getYear() - 1990; // ⚠️ Frágil: solo funciona este año
    assertThat(persona.calcularEdad()).isEqualTo(edadEsperada);
}

// BIEN: prueba repetible con fecha fija
@Test
void deberiaCalcularEdad() {
    Persona persona = new Persona("María", LocalDate.of(1990, 5, 15));
    int edadEsperada = ChronoUnit.YEARS.between(
            LocalDate.of(1990, 5, 15),
            LocalDate.of(2026, 1, 15));
    assertThat(persona.calcularEdad(LocalDate.of(2026, 1, 15))).isEqualTo(edadEsperada);
}
```

### 12.1.4 Cobertura de Código: Lo que Significa (y lo que no)

La cobertura de código mide qué porcentaje de líneas, ramas o métodos de tu código son ejecutados durante las pruebas. Pero no mide calidad.

| Métrica | Mide | No mide |
|---|---|---|
| **Cobertura de línea** | % de líneas ejecutadas al menos una vez | Si todas las combinaciones de lógica se verificaron |
| **Cobertura de rama** | % de caminos condicionales cubiertos | Si las aserciones son correctas |
| **Cobertura de método** | % de métodos invocados | Si los parámetros extremos se probaron |

```
Cobertura alta  ≠  Código bien probado
Cobertura baja  ≠  Código defectuoso
```

> [!NOTE] 🎯 **La Trampa del 100% de Cobertura:** Perseguir el 100% de cobertura de línea es un anti-patrón. Conduce a pruebas que verifican trivialidades (`getters`, `setters`) y a una falsa sensación de seguridad. **En resumen:** la cobertura es un termómetro, no un diagnóstico. Úsala para encontrar código no probado, no para certificar calidad. Profundizaremos en esto en la sección 12.11.

---

## 12.2 JUnit 5 en Profundidad

JUnit 5 es la referencia indiscutible para testing en Java. No es simplemente "JUnit 4 con esteroides": es una reescritura desde cero que introduce un modelo de extensión mucho más potente, soporte nativo para Java 8+ (lambdas, streams) y una arquitectura modular.

### 12.2.1 Arquitectura de JUnit 5

JUnit 5 se compone de tres módulos distintos:

```
┌──────────────────────────────────────────────┐
│                  JUnit 5                     │
├──────────────────────────────────────────────┤
│  ┌─────────────────────────────────────┐     │
│  │        JUnit Platform               │     │
│  │  Motor de lanzamiento. Descubre,     │     │
│  │  filtra y ejecuta tests.            │     │
│  │  API: Launcher, TestEngine          │     │
│  └──────────────┬──────────────────────┘     │
│                 │                             │
│    ┌────────────┴──────────────┐             │
│    │                           │             │
│    ▼                           ▼             │
│  ┌──────────────────┐  ┌──────────────────┐  │
│  │  Jupiter         │  │  Vintage         │  │
│  │  Nuevo modelo de │  │  Retrocompatibi- │  │
│  │  programación:   │  │  lidad con       │  │
│  │  @Test, lambda,  │  │  JUnit 3 y 4     │  │
│  │  extensibilidad  │  │                  │  │
│  └──────────────────┘  └──────────────────┘  │
└──────────────────────────────────────────────┘
```

- **JUnit Platform**: El motor fundamental. Provee la API de lanzamiento (Launcher) que descubre y ejecuta tests. IDEs, Maven, Gradle se integran con la Platform.
- **JUnit Jupiter**: El nuevo modelo de programación. Todas las anotaciones que usaremos (`@Test`, `@BeforeEach`, `@DisplayName`, etc.) pertenecen a Jupiter.
- **JUnit Vintage**: Un `TestEngine` que ejecuta tests escritos con JUnit 3 y JUnit 4, permitiendo una migración gradual.

```xml
<!-- build.gradle -->
dependencies {
    // Jupiter (nuevo modelo) + Platform (motor)
    testImplementation 'org.junit.jupiter:junit-jupiter:5.11.3'
    // Vintage (para migrar tests viejos de JUnit 4)
    testImplementation 'org.junit.vintage:junit-vintage-engine:5.11.3'
    // Platform Launcher (para ejecutar tests programáticamente)
    testRuntimeOnly 'org.junit.platform:junit-platform-launcher'
}
```

```xml
<!-- pom.xml -->
<dependency>
    <groupId>org.junit.jupiter</groupId>
    <artifactId>junit-jupiter</artifactId>
    <version>5.11.3</version>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>org.junit.vintage</groupId>
    <artifactId>junit-vintage-engine</artifactId>
    <version>5.11.3</version>
    <scope>test</scope>
</dependency>
```

### 12.2.2 La Anotación @Test

La anotación `@Test` marca un método como un caso de prueba. Desde JUnit Jupiter, `@Test` no admite atributos (a diferencia de JUnit 4, donde se especificaba `expected` y `timeout`). En JUnit 5, esas funcionalidades se implementan mediante aserciones explícitas, lo que resulta en código más legible.

```java
import org.junit.jupiter.api.Test;
import static org.junit.jupiter.api.Assertions.*;

class CalculadoraTest {

    @Test
    @DisplayName("Suma de dos números positivos debería retornar la suma correcta")
    void sumar_dosNumerosPositivos_retornaSumaCorrecta() {
        Calculadora calc = new Calculadora();
        int resultado = calc.sumar(3, 5);
        assertEquals(8, resultado, "3 + 5 debería ser 8");
    }
}
```

Convenciones de nomenclatura recomendadas (todas son válidas, elige una y sé consistente):

| Convención | Ejemplo | Ventaja |
|---|---|---|
| `snake_case_descriptivo` | `sumar_dosNumerosPositivos_retornaSuma` | Legible por humanos |
| `should_when` | `shouldReturnSum_whenTwoPositiveNumbers` | Estandarizado |
| `given_when_then` | `givenTwoPositiveNumbers_whenSum_thenCorrectResult` | Más estructurado |
| `deberiaAccion_dadoCondicion` | `deberiaSumarCorrectamente_dadoDosPositivos` | En español, natural |
| `methodName_stateUnderTest_expectedBehavior` | `sumar_numerosPositivos_resultadoCorrecto` | Técnico pero claro |

En este capítulo usaremos mayoritariamente la convención en español con formato `deberia..._dado...` y también patrones `given_when_then` en inglés cuando el contexto técnico lo amerite.

### 12.2.3 Aserciones (Assertions)

Las aserciones son el corazón de una prueba: verifican que el resultado obtenido coincide con el esperado. Si una aserción falla, la prueba falla. JUnit Jupiter ofrece un rico conjunto de métodos estáticos en `org.junit.jupiter.api.Assertions`.

#### assertEquals, assertNotEquals

```java
@Test
@DisplayName("assertEquals con tipos primitivos y objetos")
void deberiaCompararValoresCorrectamente() {
    Calculadora calc = new Calculadora();

    // Primitivos: compara por valor
    assertEquals(8, calc.sumar(5, 3));
    assertEquals(3.14, calc.obtenerPi(), 0.001); // Tolerancia para doubles

    // Objetos: usa equals()
    assertEquals(new BigDecimal("10.00"), calc.calcularImpuesto(new BigDecimal("100")));

    // Mensaje descriptivo en caso de fallo (último parámetro)
    assertEquals(42, calc.respuestaUniversal(), "La respuesta debería ser 42");

    // Verificar que dos valores NO son iguales
    assertNotEquals(0, calc.sumar(2, 2));
}
```

#### assertTrue, assertFalse

```java
@Test
@DisplayName("Afirmaciones booleanas con assertTrue y assertFalse")
void deberiaVerificarCondicionesBooleanas() {
    ValidadorEmail validador = new ValidadorEmail();

    assertTrue(validador.esValido("usuario@dominio.com"),
            () -> "El email usuario@dominio.com debería ser válido"); // Supplier para lazy evaluation

    assertFalse(validador.esValido("esto-no-es-un-email"));
}
```

#### assertNull, assertNotNull

```java
@Test
@DisplayName("Afirmaciones de nulidad con assertNull y assertNotNull")
void deberiaVerificarNulidad() {
    RepositorioUsuario repo = new RepositorioUsuario();

    Usuario encontrado = repo.buscarPorId(1L);
    assertNotNull(encontrado, "El usuario con ID 1 debería existir");

    Usuario inexistente = repo.buscarPorId(999L);
    assertNull(inexistente, "El usuario con ID 999 no debería existir");
}
```

#### assertThrows (Validación de excepciones)

En lugar del obsoleto `@Test(expected = ...)` de JUnit 4, JUnit Jupiter provee `assertThrows`:

```java
@Test
@DisplayName("Debería lanzar excepción al dividir por cero")
void deberiaLanzarExcepcion_alDividirPorCero() {
    Calculadora calc = new Calculadora();

    ArithmeticException excepcion = assertThrows(ArithmeticException.class,
            () -> calc.dividir(10, 0));

    // Podemos inspeccionar la excepción capturada
    assertEquals("/ by zero", excepcion.getMessage());
}

@Test
@DisplayName("Debería lanzar excepción con mensaje específico al validar email nulo")
void deberiaLanzarExcepcionConMensaje_alValidarEmailNulo() {
    ValidadorEmail validador = new ValidadorEmail();

    IllegalArgumentException excepcion = assertThrows(
            IllegalArgumentException.class,
            () -> validador.validar(null),
            "Debería lanzar IllegalArgumentException para email nulo"
    );

    assertTrue(excepcion.getMessage().contains("no puede ser nulo"));
}

@Test
@DisplayName("NO debería lanzar excepción con datos válidos")
void noDeberiaLanzarExcepcion_conDatosValidos() {
    Calculadora calc = new Calculadora();

    assertDoesNotThrow(() -> calc.dividir(10, 2),
            "10 / 2 no debería lanzar excepción");
}
```

#### assertAll (Agrupación de aserciones)

Cuando múltiples aserciones se aplican a un mismo objeto, `assertAll` garantiza que **todas** se ejecuten y que el reporte muestre **todos** los fallos, no solo el primero.

```java
@Test
@DisplayName("Debería crear un usuario con todas sus propiedades correctas")
void deberiaCrearUsuarioConTodasLasPropiedades() {
    Usuario usuario = new Usuario("María", "maria@correo.com", 28);

    assertAll("usuario",
            () -> assertEquals("María", usuario.getNombre(), "nombre"),
            () -> assertEquals("maria@correo.com", usuario.getEmail(), "email"),
            () -> assertEquals(28, usuario.getEdad(), "edad"),
            () -> assertTrue(usuario.esMayorDeEdad(), "mayoría de edad"),
            () -> assertFalse(usuario.getRoles().isEmpty(), "debe tener roles por defecto")
    );
}
```

Sin `assertAll`, la primera aserción fallida detendría la ejecución y nunca sabrías que también fallaron las otras tres.

#### assertTimeout y assertTimeoutPreemptively

```java
@Test
@DisplayName("Debería completar la operación en menos de 100ms")
void deberiaCompletarRapidamente() {
    ProcesadorDatos procesador = new ProcesadorDatos();

    // assertTimeout: ejecuta y verifica al final (permite que la tarea termine aunque exceda)
    String resultado = assertTimeout(
            Duration.ofMillis(100),
            () -> procesador.procesarLote(50)
    );
    assertNotNull(resultado);
}

@Test
@DisplayName("Debería abortar si tarda más de 500ms (timeout preventivo)")
void deberiaAbortarSiTardaDemasiado() {
    CalculadoraLenta calcLenta = new CalculadoraLenta();

    // assertTimeoutPreemptively: aborta la ejecución ni bien se excede el tiempo
    assertTimeoutPreemptively(
            Duration.ofMillis(500),
            () -> calcLenta.calcularFibonacci(45) // ⚠️ puede ser interrumpido
    );
}
```

> [!NOTE] ⏱️ **Diferencia clave:** `assertTimeout` ejecuta la tarea en el mismo hilo y espera a que termine; `assertTimeoutPreemptively` la ejecuta en un hilo separado y la aborta al exceder el límite. **En resumen:** usa `assertTimeout` para pruebas de rendimiento diagnóstico, y `assertTimeoutPreemptively` para código que no debería bloquearse (pero cuidado: puede dejar recursos en estado inconsistente).

#### assertArrayEquals

```java
@Test
@DisplayName("Debería comparar arrays elemento por elemento")
void deberiaCompararArraysCorrectamente() {
    Ordenador ordenador = new Ordenador();

    int[] esperado = {1, 2, 4, 5, 8};
    int[] resultado = ordenador.ordenarBurbuja(new int[]{5, 2, 8, 1, 4});

    assertArrayEquals(esperado, resultado);
}
```

#### assertIterableEquals

```java
@Test
@DisplayName("Debería comparar Iterables posición por posición")
void deberiaCompararListasCorrectamente() {
    FiltradorProductos filtrador = new FiltradorProductos();
    List<Producto> entrada = List.of(
            new Producto("A", 10.0),
            new Producto("B", 25.0),
            new Producto("C", 5.0)
    );

    List<Producto> resultado = filtrador.filtrarPorPrecioMinimo(entrada, 10.0);

    assertIterableEquals(
            List.of(new Producto("A", 10.0), new Producto("B", 25.0)),
            resultado
    );
}
```

#### Tabla resumen de aserciones

| Aserción | Propósito | Ejemplo |
|---|---|---|
| `assertEquals(expected, actual)` | Igualdad por `equals()` o `==` | Valores esperados |
| `assertNotEquals(unexpected, actual)` | Desigualdad | `assertNotEquals(0, suma)` |
| `assertTrue(condition)` | Condición verdadera | `assertTrue(lista.isEmpty())` |
| `assertFalse(condition)` | Condición falsa | `assertFalse(archivo.existe())` |
| `assertNull(value)` | Referencia nula | `assertNull(repo.buscar(999L))` |
| `assertNotNull(value)` | Referencia no nula | `assertNotNull(resultado)` |
| `assertSame(expected, actual)` | Misma referencia (`==`) | Mismo objeto en cache |
| `assertNotSame(unexpected, actual)` | Diferente referencia | Objeto clonado |
| `assertThrows(Type, executable)` | Excepción esperada | `assertThrows(IOException.class, ...)` |
| `assertDoesNotThrow(executable)` | Sin excepción | `assertDoesNotThrow(() -> ...)` |
| `assertAll(heading, executables...)` | Múltiples aserciones | Validación completa de objeto |
| `assertTimeout(duration, executable)` | No exceder tiempo | Timeout flexible |
| `assertTimeoutPreemptively(...)` | Abortar al exceder tiempo | Timeout estricto |
| `assertArrayEquals(expected, actual)` | Arrays iguales | `int[]`, `String[]` |
| `assertIterableEquals(expected, actual)` | Iterables iguales en orden | `List`, `Set` ordenado |
| `assertLinesMatch(expected, actual)` | Líneas de texto con wildcards | Output multilínea |
| `assertInstanceOf(type, object)` | Verificar tipo | `assertInstanceOf(Circulo.class, figura)` |

### 12.2.4 Asunciones (Assumptions)

Las asunciones permiten ejecutar condicionalmente una prueba. Si una asunción falla, la prueba se **aborta** (no falla), lo que es semánticamente diferente.

```java
import static org.junit.jupiter.api.Assumptions.*;

class PruebasDependientesDelEntorno {

    @Test
    @DisplayName("Solo debería ejecutarse en CI")
    void deberiaEjecutarseSoloEnCI() {
        assumeTrue("true".equals(System.getenv("CI")),
                "Abortando: no estamos en entorno CI");

        // Este código solo se ejecuta si estamos en CI
        ProcesadorBuild procesador = new ProcesadorBuild();
        assertTrue(procesador.construirReporte().length() > 0);
    }

    @Test
    @DisplayName("Solo debería ejecutarse en macOS durante desarrollo")
    void pruebaEspecificaDeMacOS() {
        assumingThat(
                System.getProperty("os.name").contains("Mac"),
                () -> {
                    // Código que solo se ejecuta en Mac
                    assertEquals('/', File.separatorChar);
                }
        );

        // Esto se ejecuta SIEMPRE, independientemente del SO
        Calculadora calc = new Calculadora();
        assertEquals(4, calc.sumar(2, 2));
    }
}
```

| Método | Comportamiento |
|---|---|
| `assumeTrue(condition)` | Aborta si `condition == false` |
| `assumeFalse(condition)` | Aborta si `condition == true` |
| `assumingThat(condition, executable)` | Ejecuta `executable` solo si `condition` es true, pero no aborta si es false |
| `assumeTrue(condition, message)` | Aborta con mensaje personalizado |
| `assumeTrue(condition, messageSupplier)` | Mensaje con supplier para lazy evaluation |

---

[← Capítulo anterior](capitulo-11-spring-boot.md) | [Inicio](../README.md) | [Capítulo siguiente →](capitulo-13-maven-gradle.md)
