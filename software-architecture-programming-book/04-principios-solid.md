# Capítulo 4: Principios SOLID — La Diferencia entre Código que Funciona y Código que Perdura

> "Cualquiera puede escribir código que una máquina entienda. Solo los buenos escriben código que un humano pueda mantener." — Martin Fowler

## 4.1 ¿Qué es SOLID y Por Qué Debería Importarte?

SOLID son cinco principios de diseño orientado a objetos formulados por Robert C. Martin (Uncle Bob) a principios de los 2000. Pero no te equivoques: esto no es teoría académica para pasar exámenes. Es la diferencia entre un sistema que en 2 años todavía se puede modificar y uno que el equipo odia con pasión.

**La promesa de SOLID**: si sigues estos principios, tu código será fácil de cambiar. Y el software que no se puede cambiar está muerto.

**La advertencia**: SOLID es un medio, no un fin. Seguirlo ciegamente produce sobre-ingeniería. Ignorarlo produce Big Ball of Mud. La maestría está en saber cuándo aplicarlo, cuándo flexibilizarlo y cuándo romperlo conscientemente.

```
Imagina dos codebases. Ambas hacen lo mismo. Misma funcionalidad.
Mismos tests pasando. Pero:

Codebase A (sin SOLID):
  • Cambiar "email" por "correo" en una pantalla requiere tocar 47 archivos.
  • El onboarding de un dev nuevo toma 4 meses.
  • Cada sprint: 40% features nuevas, 60% apagar incendios.
  • El equipo rota porque nadie quiere mantener esto.

Codebase B (con SOLID):
  • Cambiar "email" por "correo" es 1 archivo de vista + 1 de validación.
  • Un dev nuevo es productivo en 3 semanas.
  • Cada sprint: 80% features, 20% mejora continua.
  • El equipo se queda porque el código es un placer.

Esa es la diferencia. No es académica. Es supervivencia profesional.
```

---

## 4.2 SRP — Single Responsibility Principle (Principio de Responsabilidad Única)

### La Definición que Todos Repiten (y Está Incompleta)

> "Una clase debe tener una, y solo una, razón para cambiar."

Pero ¿qué significa "razón para cambiar"? No significa "hacer una sola cosa". Significa: **una clase debe tener un solo actor que solicite cambios**.

### El Error Más Común

Casi todos los desarrolladores interpretan SRP como "mi clase debe ser pequeña y hacer poquitas cosas". Eso no es SRP, es sentido común. SRP es más profundo: es sobre **personas y roles en la organización**.

```
Imagina esta clase en un sistema de RRHH:

class Empleado {
    // Datos que le importan a RRHH
    String nombre;
    String direccion;
    BigDecimal salario;
    String categoria; // Junior, Senior, Manager...

    // Esto lo usa RRHH para pagar
    BigDecimal calcularSalario() {
        if (categoria.equals("Junior")) return salario;
        if (categoria.equals("Senior")) return salario * 1.5;
        if (categoria.equals("Manager")) return salario * 2.0;
        return salario;
    }

    // Esto lo usa Operaciones para asignar turnos
    String generarHorarioSemanal() {
        // Lógica de horarios...
    }

    // Esto lo usa el DBA para persistir
    void guardarEnBaseDeDatos() {
        // INSERT INTO empleados...
    }

    // Esto lo usa Finanzas para reportes
    String generarReporteFinanciero() {
        // Reporte para el CFO...
    }
}
```

¿Ves el problema? Esta clase cambiará cuando:
- RRHH cambie la política salarial.
- Operaciones cambie el formato de horarios.
- El DBA migre de PostgreSQL a MongoDB.
- Finanzas pida un nuevo campo en el reporte.

**Cuatro actores diferentes, cuatro razones para cambiar. Una sola clase.** Esto es una bomba de tiempo. Cuando RRHH pide un cambio y accidentalmente rompes la lógica de Finanzas porque comparten la misma clase, tienes un problema grave.

### El Refactor que Salva Vidas

```java
// ─── Cada responsabilidad en su propia clase ───

// Esto es lo único que le importa a RRHH
class DatosEmpleado {
    private String id;
    private String nombre;
    private String direccion;
    private CategoriaEmpleado categoria;
    private BigDecimal salarioBase;
    // Solo getters, sin lógica
}

// RRHH cambia esto sin afectar a nadie más
class CalculadoraSalarial {
    BigDecimal calcular(DatosEmpleado empleado) {
        return switch (empleado.getCategoria()) {
            case JUNIOR -> empleado.getSalarioBase();
            case SENIOR -> empleado.getSalarioBase().multiply(new BigDecimal("1.5"));
            case MANAGER -> empleado.getSalarioBase().multiply(new BigDecimal("2.0"));
        };
    }
}

// Operaciones cambia esto sin afectar a RRHH
class PlanificadorHorarios {
    HorarioSemanal generar(DatosEmpleado empleado, Semana semana) {
        // Lógica de horarios aquí
    }
}

// DBA cambia la implementación sin que nadie se entere
interface EmpleadoRepository {
    void guardar(DatosEmpleado empleado);
    Optional<DatosEmpleado> buscarPorId(String id);
}

class PostgresEmpleadoRepository implements EmpleadoRepository {
    void guardar(DatosEmpleado empleado) {
        // INSERT ...
    }
}

class MongoEmpleadoRepository implements EmpleadoRepository {
    void guardar(DatosEmpleado empleado) {
        // db.empleados.insertOne(...)
    }
}

// Finanzas tiene su propio mundo
class GeneradorReporteFinanciero {
    ReporteFinanciero generar(List<DatosEmpleado> empleados) {
        // Lógica financiera aislada
    }
}
```

### La Señal de Alarma que Debes Reconocer

¿Cómo sabes que estás violando SRP en producción? Busca estos síntomas:

```
🔴 Clases con más de 200-300 líneas.
   No es una regla estricta, pero si cada clase tiene 400+ líneas,
   casi seguro que está haciendo demasiado.

🔴 Métodos con "y" o "and" en el nombre.
   "calcularSalarioYGuardar()" → dos responsabilidades.
   "validarYEnviar()" → divídelo.

🔴 Cuando preguntas "¿quién pidió este cambio?" y la respuesta es
   "tres departamentos diferentes pidieron cambios en la misma clase
   este mes" → SRP violado.

🔴 Tests que se rompen por razones no relacionadas con lo que testean.
   "Cambié la política de descuentos y 40 tests de facturación fallaron."
   → El código de descuentos está acoplado al de facturación.

🔴 Merge conflicts constantes en la misma clase.
   "Juan y María siempre editan la misma clase en cada sprint."
   → Esa clase es un imán de cambios porque tiene demasiadas responsabilidades.
```

### La Pregunta que Debes Hacerte

Cada vez que escribas una clase, pregúntate: **"¿Quién va a pedir cambios en esta clase?"** Si la respuesta es más de una persona/rol/departamento, probablemente necesitas dividir.

---

## 4.3 OCP — Open/Closed Principle (Principio Abierto/Cerrado)

### La Definición que Engaña

> "Las entidades de software deben estar abiertas para extensión, pero cerradas para modificación."

Suena contradictorio, ¿no? ¿Cómo extiendo sin modificar?

La respuesta: **diseñas puntos de extensión por adelantado y luego extiendes sin tocar el código existente**. Es como un enchufe eléctrico: no necesitas modificar la instalación de tu casa para conectar un electrodoméstico nuevo.

### El Dolor que OCP Evita

Imagina que trabajas en una pasarela de pagos. Hoy solo aceptas tarjeta de crédito. El código es simple:

```java
// ❌ ESTO VA A DOLER — y te lo digo por experiencia
class ProcesadorPagos {
    String procesar(Pago pago) {
        if (pago.getMetodo().equals("TARJETA_CREDITO")) {
            // 300 líneas de integración con Stripe
            Stripe.apiKey = "sk_live_xxx";
            Stripe.Charge charge = Stripe.Charge.create(...);
            // validar, reintentar, loguear, manejar errores...
            return charge.getId();
        }
        throw new RuntimeException("Método no soportado");
    }
}
```

Semana 1: El CEO llega. "Necesitamos PayPal. Para el viernes."

Entonces haces esto:

```java
class ProcesadorPagos {
    String procesar(Pago pago) {
        if (pago.getMetodo().equals("TARJETA_CREDITO")) {
            // 300 líneas de Stripe que YA FUNCIONAN
            // Rezas no romper nada al añadir código
        } else if (pago.getMetodo().equals("PAYPAL")) {
            // 250 líneas de integración con PayPal
            // Copiaste y pegaste patrones de Stripe
            // Pero PayPal maneja errores diferente...
        }
        throw new RuntimeException("Método no soportado");
    }
}
```

Semana 3: "Necesitamos MercadoPago para Brasil."

Vuelves a tocar ProcesadorPagos. Ya tienes 550 líneas de if/else. Los tests de Stripe y PayPal se rompen porque tocaste la misma clase. Estás rezando un rosario cada vez que desplegas.

Semana 8: "Pix para Brasil." Ya tienes 800 líneas. La clase es inmantenible. Nadie en el equipo quiere tocarla. Es "esa clase".

**Esto es real. Me pasó. No te miento.**

### El Refactor que Te Hace Dormir Tranquilo

```java
// ─── Paso 1: Define el contrato (punto de extensión) ───
interface PasarelaPago {
    ResultadoPago procesar(SolicitudPago solicitud);
    String nombre(); // Para logs y debugging
}

// ─── Paso 2: Cada pasarela en su propio mundo ───
class StripePasarela implements PasarelaPago {
    @Override
    public ResultadoPago procesar(SolicitudPago solicitud) {
        // 300 líneas de Stripe. Si hay un bug, es SOLO aquí.
        // No afecta a PayPal. No afecta a MercadoPago.
    }
    
    @Override
    public String nombre() { return "Stripe"; }
}

class PayPalPasarela implements PasarelaPago {
    @Override
    public ResultadoPago procesar(SolicitudPago solicitud) {
        // 250 líneas de PayPal. Vive en su propio archivo.
        // Los cambios aquí jamás rompen Stripe.
    }
    
    @Override
    public String nombre() { return "PayPal"; }
}

class MercadoPagoPasarela implements PasarelaPago {
    @Override
    public ResultadoPago procesar(SolicitudPago solicitud) {
        // Integración con API de MercadoPago.
    }
    
    @Override
    public String nombre() { return "MercadoPago"; }
}

// ─── Paso 3: El orquestador no sabe qué pasarela usa ───
class ServicioPagos {
    private final List<PasarelaPago> pasarelas;

    ServicioPagos(List<PasarelaPago> pasarelas) {
        this.pasarelas = pasarelas;
    }

    ResultadoPago procesar(SolicitudPago solicitud) {
        PasarelaPago pasarela = pasarelas.stream()
            .filter(p -> p.nombre().equals(solicitud.getMetodoPago()))
            .findFirst()
            .orElseThrow(() -> new MetodoNoSoportado(solicitud.getMetodoPago()));
        
        // Logging, métricas, auditoría — esto NO cambia nunca
        log.info("Procesando pago con {}", pasarela.nombre());
        ResultadoPago resultado = pasarela.procesar(solicitud);
        metrica.registrarPago(pasarela.nombre(), resultado);
        return resultado;
    }
}
```

Ahora añadir un nuevo método de pago es: crear una clase nueva. Punto. No tocas Stripe. No tocas PayPal. No tocas MercadoPago. **Cero riesgo de romper lo que ya funciona.**

```
Antes: Añadir pasarela = modificar clase existente + riesgo de romper todo.
Ahora: Añadir pasarela = crear archivo nuevo  + riesgo cero.
```

Esto es OCP en acción. Y créeme: cuando tienes 8 pasarelas de pago en 5 países, este principio no es opcional. Es supervivencia.

### Más Allá de los Pagos — OCP Está en Todas Partes

```
El mismo patrón aplica para:

• Validaciones de negocio:
  interface ReglaNegocio { boolean evaluar(Pedido p); String mensajeError(); }
  Cada regla es una clase. Añadir regla = clase nueva.

• Notificaciones:
  interface Notificador { void enviar(Mensaje m); }
  EmailNotificador, SMSNotificador, PushNotificador, SlackNotificador...

• Formatos de exportación:
  interface FormatoExportador { byte[] exportar(Datos d); }
  PDFExportador, CSVExportador, ExcelExportador...

• Proveedores de envío:
  interface Transportista { Envio crearEnvio(Paquete p); }
  FedexTransportista, DHLTransportista, CorreosTransportista...
```

### La Trampa del OCP Prematuro

> "No abstraigas para un futuro que quizá nunca llegue." — YAGNI

El error opuesto es igual de grave: crear abstracciones "por si acaso" para extensiones que nunca ocurren.

```java
// ❌ SOBRE-INGENIERÍA: Solo tienes UNA pasarela de pago hoy.
// No crees una fábrica abstracta con strategy, factory method,
// builder y dependency injection para "el futuro".
class ProcesadorPagos {
    StripePasarela stripe;
    
    ResultadoPago procesar(SolicitudPago s) {
        return stripe.procesar(s); // Simple. Funciona. Suficiente.
    }
}

// Cuando llegue la SEGUNDA pasarela, AHÍ sí aplicas OCP.
// No antes. El truco: diseña de forma que sea FÁCIL aplicar OCP
// cuando lo necesites, pero no lo implementes hasta que duela.
```

---

## 4.4 LSP — Liskov Substitution Principle (Principio de Sustitución de Liskov)

### La Definición que Nadie Entiende la Primera Vez

> "Los objetos de una clase derivada deben poder sustituir a objetos de la clase base sin alterar el correcto funcionamiento del programa."

Traducción humana: si tienes una función que recibe un `Animal`, debería funcionar igual si le pasas un `Perro`, un `Gato` o un `Pájaro`. Si pasarle un `Pingüino` rompe todo porque no vuela, violaste LSP.

### El Ejemplo Más Famoso (y Por Qué Importa)

```java
// ❌ LA TRAMPA CLÁSICA QUE TODO DESARROLLADOR HA CAÍDO

class Rectangulo {
    protected int ancho;
    protected int alto;
    
    void setAncho(int ancho) { this.ancho = ancho; }
    void setAlto(int alto) { this.alto = alto; }
    
    int getAncho() { return ancho; }
    int getAlto() { return alto; }
    int area() { return ancho * alto; }
}

class Cuadrado extends Rectangulo {
    @Override
    void setAncho(int ancho) {
        this.ancho = ancho;
        this.alto = ancho; // Mantiene la invariante del cuadrado
    }
    
    @Override
    void setAlto(int alto) {
        this.alto = alto;
        this.ancho = alto; // Mantiene la invariante del cuadrado
    }
}
```

Hasta aquí todo parece lógico. Un cuadrado ES UN rectángulo. La herencia parece correcta.

**Ahora viene el dolor:**

```java
// Esta función existe en 50 lugares de tu codebase
void redimensionarRectangulo(Rectangulo rectangulo) {
    rectangulo.setAncho(5);
    rectangulo.setAlto(10);
    // El programador espera que el área sea 5 × 10 = 50
    assert rectangulo.area() == 50;
}

// Funciona con Rectangulo:
Rectangulo r = new Rectangulo();
redimensionarRectangulo(r); // ✓ área = 50

// Pero con Cuadrado:
Cuadrado c = new Cuadrado();
redimensionarRectangulo(c); // ✗ área = 100 (porque alto se puso a 10
                            //    y ancho también cambió a 10)
```

¿Qué pasó? `Cuadrado` violó LSP. No puede sustituir a `Rectangulo` sin romper código que depende del comportamiento esperado de `Rectangulo` (que `setAncho` no afecta a `alto`).

### El Problema No Es el Cuadrado. Es la Abstracción Incorrecta.

```
La herencia que viola LSP te dice:
  "Un cuadrado ES UN rectángulo" — Matemáticamente sí.
  "Un Cuadrado PUEDE SUSTITUIR a Rectangulo" — En software, NO.

La diferencia es sutil pero mortal:
  La herencia en OOP no es clasificación taxonómica.
  Es CONTRATO DE COMPORTAMIENTO.
```

### Ejemplo Real que Rompe Sistemas en Producción

```java
// Un sistema de archivos — ejemplo que vi en un sistema real

interface Archivo {
    byte[] leer();
    void escribir(byte[] datos);
    long tamaño();
}

class ArchivoNormal implements Archivo {
    // Implementación normal. Funciona perfecto.
}

class ArchivoComprimido implements Archivo {
    private byte[] datosComprimidos;
    
    @Override
    byte[] leer() {
        return descomprimir(datosComprimidos); // ✓ Funciona
    }
    
    @Override
    void escribir(byte[] datos) {
        datosComprimidos = comprimir(datos); // ✓ Funciona
    }
    
    @Override
    long tamaño() {
        // ❌ ¿Qué devuelvo? ¿Tamaño comprimido o descomprimido?
        // Si devuelvo comprimido: el sistema asigna mal el espacio.
        // Si devuelvo descomprimido: tengo que descomprimir TODO
        //    (posiblemente GBs de datos) solo para saber el tamaño.
        //
        // CUALQUIER implementación viola expectativas de alguien.
        // ArchivoComprimido NO puede sustituir a Archivo.
    }
}
```

Este es el tipo de bug que no encuentras en desarrollo. Lo encuentras cuando el sistema de archivos asigna mal el espacio porque `tamaño()` devolvió el tamaño comprimido y el sistema de quotas bloqueó a un usuario que no debía. O cuando `tamaño()` descomprime 2 GB de datos solo para reportar un número en una UI que se refresca cada 30 segundos.

### Cómo Evitar Violar LSP

```
1. Prefiere composición sobre herencia.
   En lugar de "Cuadrado extends Rectangulo", crea una interfaz
   "Figura" con area() y que cada figura tenga sus propias reglas.

2. Documenta el CONTRATO, no solo la API.
   Java: @param ancho — NO modifica el alto. (Si lo hace, documéntalo.)
   
3. Si necesitas que el comportamiento varíe, hazlo explícito.
   En vez de ArchivoComprimido implements Archivo:
   interface Archivo { byte[] leer(); }
   interface ArchivoConEscritura extends Archivo { void escribir(byte[] d); }
   // Así no fuerzas a ArchivoComprimido a implementar métodos que no puede honrar.

4. Pregúntate: ¿puedo pasar una instancia de esta subclase a CUALQUIER
   método que acepte la clase base sin que el programa falle?
   Si la respuesta es "depende" o "no" → violaste LSP.

5. Los tests de la clase base deben pasar con TODAS las subclases.
   Si tienes 20 tests para Rectangulo y Cuadrado pasa 18, tienes un problema.
```

---

## 4.5 ISP — Interface Segregation Principle (Principio de Segregación de Interfaces)

### El Principio que Nadie Respeta Hasta que Sufre

> "Ningún cliente debe verse forzado a depender de métodos que no usa."

En español: **no me obligues a implementar cosas que no necesito.**

### La Interfaz Monolítica — El Pecado Original

```java
// ❌ ESTA INTERFAZ ES EL DIABLO — la he visto en producción decenas de veces
interface Repositorio<T> {
    T guardar(T entidad);
    Optional<T> buscarPorId(String id);
    List<T> buscarTodos();
    List<T> buscarConFiltros(Map<String, String> filtros);
    List<T> buscarConPaginacion(int pagina, int tamaño);
    void eliminar(String id);
    void eliminarTodos();
    long contar();
    boolean existe(String id);
    T actualizar(T entidad);
    List<T> guardarVarios(List<T> entidades);
    // ... y siguen apareciendo métodos cada sprint
}
```

"Pero es genérica, ¡sirve para todo!" **Ese es exactamente el problema.** Sirve para todo y por lo tanto es una pesadilla para todos los que la implementan.

```
Imagina que tienes que implementar esta interfaz para:

1. Usuarios (CRUD completo): ✓ Usa casi todos los métodos.
2. Configuración del sistema (solo lectura): 
   ✗ Tengo que implementar guardar(), eliminar(), eliminarTodos()...
     Lanzan UnsupportedOperationException. Eso es feo, inseguro,
     y cualquier dev nuevo puede llamarlos sin saber que rompen.

3. Caché en Redis (solo guardar y buscarPorId):
   ✗ 9 métodos que lanzan UnsupportedOperationException.
   
4. Eventos de auditoría (solo guardar, nunca leer):
   ✗ 7 métodos inútiles que igual tengo que compilar y testear.
```

### El Refactor que Te Hace Querer Programar de Nuevo

```java
// ✅ Interfaces pequeñas y específicas. Cada cliente implementa SOLO lo que usa.

interface Lector<T> {
    Optional<T> buscarPorId(String id);
    List<T> buscarTodos();
    List<T> buscarConFiltros(Filtros filtros);
    List<T> buscarConPaginacion(Paginacion paginacion);
    long contar();
    boolean existe(String id);
}

interface Escritor<T> {
    T guardar(T entidad);
    List<T> guardarVarios(List<T> entidades);
    T actualizar(T entidad);
    void eliminar(String id);
    void eliminarTodos();
}

// Ahora cada servicio implementa lo que REALMENTE necesita:

// Configuración del sistema: solo lectura
class ConfiguracionRepositorio implements Lector<Configuracion> {
    // Implemento 6 métodos que realmente uso.
    // Sin UnsupportedOperationException. Sin métodos vacíos.
    // El compilador me protege: no puedo llamar a guardar() por accidente.
}

// Caché Redis: lectura y escritura
class RedisCacheRepositorio implements Lector<Producto>, Escritor<Producto> {
    // Solo los 8 métodos que necesito. Clara intención.
}

// Auditoría: solo escritura
class AuditoriaRepositorio implements Escritor<EventoAuditoria> {
    // Solo guardar. No me obligas a implementar lecturas que nunca usaré.
}

// CRUD completo
class UsuarioRepositorio implements Lector<Usuario>, Escritor<Usuario> {
    // Todos los métodos. Porque los necesito.
}
```

### ISP en el Mundo Real — Más Allá de los Repositorios

```
Situación real: Microservicio de Notificaciones versión 1.

interface ServicioNotificaciones {
    void enviarEmail(Email email);
    void enviarSMS(SMS sms);
    void enviarPush(PushNotification push);
    void enviarSlack(SlackMessage msg);
    void enviarWhatsApp(WhatsAppMessage msg);
}

Servicio de Pedidos usa: solo enviarEmail y enviarSMS.
Servicio de Chat usa: solo enviarPush.
Servicio de Monitoreo usa: solo enviarSlack.

Cada servicio depende de 5 métodos, usa 1 o 2.
Cuando enviamos una nueva versión de la librería ServicioNotificaciones
porque añadimos "enviarWhatsApp", TODOS los servicios deben recompilar
y redesplegar aunque no usen WhatsApp.

Solución ISP:
interface NotificadorEmail { void enviar(Email e); }
interface NotificadorSMS { void enviar(SMS s); }
interface NotificadorPush { void enviar(PushNotification p); }
interface NotificadorSlack { void enviar(SlackMessage s); }

Cada servicio importa SOLO la interfaz que usa.
Añadir WhatsApp → solo lo importa el servicio de WhatsApp.
Cero redespliegues innecesarios. Cero acoplamiento fantasma.
```

### ISP y el Acoplamiento Fantasma

El verdadero peligro de violar ISP no es tener métodos vacíos. Es el **acoplamiento transitivo de dependencias**:

```
Servicio A usa la interfaz gigante IRepositorio.
IRepositorio depende de javax.persistence.EntityManager.
javax.persistence depende de hibernate-core.
hibernate-core depende de 30 jars más.

Pero Servicio A SOLO implementa buscarPorId() y contar().
Nunca usa EntityManager. Nunca usa Hibernate.

Sin embargo, su classpath tiene 30+ jars innecesarios.
Su build tarda más. Su imagen Docker pesa más.
Su superficie de ataque es mayor (vulnerabilidades en jars que ni usa).
```

Esto no es teoría. He visto equipos reduciendo builds de 8 minutos a 90 segundos y imágenes Docker de 600 MB a 200 MB solo aplicando ISP a sus dependencias.

---

## 4.6 DIP — Dependency Inversion Principle (Principio de Inversión de Dependencias)

### El Principio Más Poderoso (y el Menos Entendido)

> "Depende de abstracciones, no de implementaciones concretas."

Parece simple. Pero su poder real está en la **inversión de la dirección de dependencia**.

### El Diagrama que Lo Cambia Todo

```
Arquitectura tradicional (sin DIP):

┌──────────────┐     ┌──────────────────┐     ┌──────────────┐
│  Lógica de   │────►│ Repositorio      │────►│  PostgreSQL   │
│  Negocio     │     │ PostgreSQL       │     │  Driver       │
│  (alto nivel)│     │ (bajo nivel)     │     │  (bajo nivel) │
└──────────────┘     └──────────────────┘     └──────────────┘

La flecha apunta hacia abajo. La lógica de negocio depende
de la base de datos. Cambiar PostgreSQL por MongoDB = reescribir
la lógica de negocio. ¿Te suena familiar?


Arquitectura con DIP:

┌──────────────┐          ┌──────────────────┐
│  Lógica de   │─────────►│   Repositorio    │◄────────┐
│  Negocio     │  (usa)   │   (interfaz)     │         │
│  (alto nivel)│          │   en DOMINIO     │         │
└──────────────┘          └──────────────────┘         │
                               ▲                       │
                               │                       │
                          ┌────┴─────────┐    ┌────────┴────────┐
                          │PostgresRepo  │    │   MongoRepo     │
                          │(implementa)  │    │   (implementa)  │
                          └──────────────┘    └─────────────────┘

AMBAS flechas apuntan hacia la interfaz (abstracción).
La lógica de negocio NO sabe si usa Postgres o Mongo.
```

Esto es la inversión: en lugar de que el alto nivel dependa del bajo nivel, **ambos dependen de una abstracción**. Y la abstracción la define el alto nivel (el dominio), no el bajo nivel (la infraestructura).

### El Poder Real del DIP — Testabilidad

```java
// ❌ Sin DIP: código imposible de testear sin BD real
class ServicioNotificacion {
    private final EmailSender emailSender = new EmailSender(
        "smtp.gmail.com", 587, "user", "pass"
    );
    
    void notificarPedidoConfirmado(Pedido pedido) {
        String cuerpo = "Tu pedido #" + pedido.getId() + " está confirmado.";
        emailSender.enviar(pedido.getEmail(), "Pedido Confirmado", cuerpo);
    }
}

// ¿Cómo pruebas esto?
// 1. ¿Envías emails de verdad en cada test? (lento, $$$, frágil)
// 2. ¿Mockeas EmailSender? (no puedes, está instanciado con new)
// 3. ¿PowerMock para mockear el constructor? (oscuro, frágil, legacy)
// RESULTADO: el equipo no escribe tests. "Es muy difícil."


// ✅ Con DIP: test trivial, rápido y confiable
interface Notificador {
    void enviar(String destinatario, String asunto, String cuerpo);
}

class EmailNotificador implements Notificador {
    private final EmailSender sender;
    
    @Override
    void enviar(String dest, String asunto, String cuerpo) {
        sender.enviar(dest, asunto, cuerpo);
    }
}

class ServicioNotificacion {
    private final Notificador notificador;
    
    // Inyección por constructor — el que usa la clase no decide
    // qué implementación recibe. Se la pasan de afuera.
    ServicioNotificacion(Notificador notificador) {
        this.notificador = notificador;
    }
    
    void notificarPedidoConfirmado(Pedido pedido) {
        String cuerpo = "Tu pedido #" + pedido.getId() + " está confirmado.";
        notificador.enviar(pedido.getEmail(), "Pedido Confirmado", cuerpo);
    }
}

// Test: simple, rápido, sin emails reales
@Test
void notificaPedidoConfirmado() {
    Notificador mockNotificador = mock(Notificador.class);
    ServicioNotificacion servicio = new ServicioNotificacion(mockNotificador);
    
    Pedido pedido = new Pedido("123", "cliente@email.com");
    servicio.notificarPedidoConfirmado(pedido);
    
    verify(mockNotificador).enviar(
        "cliente@email.com",
        "Pedido Confirmado",
        contains("123")
    );
}
```

### DIP No Es Solo Dependency Injection

Mucha gente confunde DIP con Dependency Injection (DI). No son lo mismo:

```
DIP (principio):   "Depende de abstracciones, no de concreciones."
                   Es el QUÉ. Es una decisión de diseño.

DI (patrón):       "Alguien externo te pasa tus dependencias."
                   Es el CÓMO. Es una técnica de implementación.

Puedes tener DI sin DIP:
  class Servicio {
      @Inject PostgreSQLRepository repo; // DI, pero dependes de concreción
  }

Puedes tener DIP sin DI:
  class Servicio {
      Repositorio repo = RepositorioFactory.crear(); // DIP, sin DI
  }

Lo ideal: DIP + DI. Pero entiende la diferencia.
```

### La Regla de Oro del DIP

> **El código que cambia por razones de negocio (dominio) NUNCA debe depender de código que cambia por razones técnicas (infraestructura).**

```
Dominio:  Cambia porque el negocio evoluciona.
          "Ahora los pedidos requieren aprobación del manager."
          → Cambia Pedido, ConfirmarPedidoUseCase.

Infra:    Cambia porque la tecnología evoluciona.
          "Migramos de PostgreSQL a MongoDB."
          "Actualizamos Spring Boot 2 a 3."
          → Cambia PostgresPedidoRepository, pero NUNCA Pedido.
```

Si tu clase `Pedido` importa algo de `javax.persistence`, ya violaste DIP. Si tu `ConfirmarPedidoUseCase` sabe que los datos vienen de PostgreSQL, ya violaste DIP.

---

## 4.7 SOLID en Conjunto — Cómo se Refuerzan Mutuamente

Los cinco principios no son independientes. Se potencian:

```
SRP + DIP:   Si separas responsabilidades, es más fácil invertir dependencias.
             Una clase con una responsabilidad tiene una dependencia clara.

OCP + DIP:   DIP es el habilitador de OCP. Sin abstracciones (DIP)
             no puedes extender sin modificar (OCP).

LSP + DIP:   Si tu abstracción (interfaz) no puede ser sustituida (LSP)
             por sus implementaciones, DIP no sirve de nada.

ISP + SRP:   Interfaces segregadas = clientes con necesidades específicas.
             Eso fuerza a las clases a tener responsabilidades claras.
```

---

## 4.8 SOLID No Es un Dogma — Cuándo Romper Cada Principio

| Principio | Cuándo Romperlo |
|-----------|----------------|
| **SRP** | Microservicios pequeños (50 líneas) donde separar sería sobre-ingeniería. Prototipos que vivirán 2 semanas. |
| **OCP** | Cuando solo tienes UNA implementación. No abstraigas "por si acaso". YAGNI > OCP. |
| **LSP** | Cuando la herencia modela correctamente la relación del mundo real Y documentas el contrato explícitamente. |
| **ISP** | Clientes internos en un monolito donde la recompilación no cuesta. APIs que solo tú consumes. |
| **DIP** | Objetos inmutables de dominio (Value Objects) que son estables. No necesitan interfaz. `new Money(10, "USD")` está bien. |

---

## 4.9 Ejercicio — Refactoriza Esta Clase Real

```java
// Esta clase existe en un sistema de facturación electrónica que heredaste.
// Tiene 700 líneas. Nadie la quiere tocar. Cada cambio rompe algo.

class Facturador {
    void facturar(Pedido pedido) {
        // 1. Validar pedido (100 líneas)
        if (pedido.getCliente() == null) throw ...
        if (pedido.getItems().isEmpty()) throw ...
        // validaciones de negocio, de formato, de integridad...
        
        // 2. Calcular impuestos (150 líneas)
        if (pedido.getPais().equals("MX")) {
            // IVA México 16%
        } else if (pedido.getPais().equals("CO")) {
            // IVA Colombia 19%
        } else if ...
        // 10 países más, cada uno con reglas distintas
        
        // 3. Generar PDF (200 líneas)
        // Apache PDFBox, layout manual...
        
        // 4. Generar XML para el SAT/hacienda (200 líneas)
        // XML con namespaces, firmas digitales...
        
        // 5. Enviar por email (50 líneas)
        // JavaMail, SMTP config...
        
        // 6. Guardar en BD (50 líneas)
        // JDBC directo
    }
}
```

**Tu tarea**: Aplica SOLID a esta clase. ¿Qué interfaces creas? ¿Qué responsabilidades separas? ¿Cómo harías para añadir soporte para Perú (nuevo país, nuevas reglas de impuestos, nuevo formato XML) sin tocar el código existente?

---

> **Reflexión del capítulo**: SOLID no es una lista de reglas para pasar entrevistas. Es el conjunto de principios que, aplicados con criterio, separan el código desechable del código que sobrevive 10 años. La diferencia no la ves en el primer mes. La ves en el mes 18, cuando el equipo sigue entregando features al mismo ritmo que el primer día. Eso es SOLID. No perfección académica. Supervivencia profesional.

---

← [Capítulo anterior](03-atributos-calidad.md) | [Inicio](README.md) | [Capítulo siguiente →](05-principios-diseno.md)
