# Capítulo 5: Principios de Diseño Esenciales — El Arte de Escribir Código que No Da Vergüenza

> "La simplicidad es el requisito más difícil de lograr." — Edsger Dijkstra

## 5.0 ¿Por Qué Principios y No Solo SOLID?

SOLID te enseña a estructurar clases y dependencias. Pero hay decisiones de diseño que SOLID no cubre: ¿deberías crear esta abstracción o es YAGNI? ¿Este código está duplicado o solo es coincidencia? ¿Esto es simple o es simplón?

Estos principios son las herramientas finas del oficio. No los aprendes en un bootcamp. Los aprendes cuando has mantenido suficiente código ajeno como para saber exactamente qué duele y por qué.

---

## 5.1 DRY — Don't Repeat Yourself (No Te Repitas)

### El Principio Más Malinterpretado de la Historia

> "Cada pieza de conocimiento debe tener una representación única, inequívoca y autorizada en el sistema." — Andy Hunt y Dave Thomas, The Pragmatic Programmer, 1999

Casi todos creen que DRY significa "no repitas código". **Error.** DRY significa "no repitas **conocimiento**". La diferencia es abismal y confundirla causa más daño que no aplicar DRY en absoluto.

### Cuando Dos Códigos Idénticos NO Son Duplicación

```python
# Imagina que trabajas en un e-commerce
# Estas dos funciones se ven casi idénticas:

def validar_email_cliente(email: str) -> bool:
    if not email:
        return False
    if "@" not in email:
        return False
    if len(email) > 254:
        return False
    return True

def validar_email_soporte(email: str) -> bool:
    if not email:
        return False
    if "@" not in email:
        return False
    if len(email) > 254:
        return False
    return True
```

Un desarrollador junior ve esto y grita "¡DRY! ¡Está duplicado!" y crea:

```python
# ❌ MAL: Unificó código pero DESTRUYÓ el conocimiento
def validar_email(email: str) -> bool:
    if not email:
        return False
    if "@" not in email:
        return False
    if len(email) > 254:
        return False
    return True
```

**Tres meses después:**

- Producto decide que los emails de soporte pueden aceptar formatos internos tipo "agente-7@interno". La validación de soporte necesita cambiar.
- Compliance exige que los emails de cliente validen contra una lista de dominios prohibidos. La validación de cliente necesita cambiar.

Ambos cambios tocan la MISMA función. Cada cambio rompe al otro equipo. Estás en un infierno de merge conflicts y bugs cruzados.

**¿Por qué falló DRY aquí?** Porque el código era idéntico pero el **conocimiento** era diferente. "Cómo se valida el email de un cliente" y "cómo se valida el email de soporte" son dos piezas de conocimiento distintas que casualmente tienen la misma implementación hoy. Mañana divergen.

### La Pregunta Correcta para DRY

No preguntes: "¿Este código es igual a este otro?"

Pregunta: **"Si esto cambia, ¿debería cambiar lo otro también?"**

```
✅ Debes unificar si:
   "Cada vez que cambie la fórmula del IVA, quiero que cambie en todas partes."
   → Extrae la fórmula del IVA a un solo lugar.

❌ No unifiques si:
   "Validar email de cliente y validar email de soporte son reglas
    de negocio diferentes que hoy casualmente son iguales."
   → Mantenlas separadas aunque el código sea idéntico.
```

### El Verdadero Peligro de DRY Mal Aplicado

```
Escenario real que me costó un finde de debugging:

Un sistema de facturación con 3 módulos: Argentina, México, Colombia.

Función calcular_impuesto(pais, monto):
    if pais == "AR": return monto * 0.21   # IVA Argentina
    if pais == "MX": return monto * 0.16   # IVA México
    if pais == "CO": return monto * 0.19   # IVA Colombia

"DRY!" — dijo el dev. Y refactorizó:

Base de datos: tabla "paises" con columna "iva"
Función calcular_impuesto(pais, monto):
    return monto * db.get_iva(pais)

¡Perfecto! Añadir un país ahora es un INSERT, no un deploy.

TRES SEMANAS DESPUÉS:

México cambia su IVA del 16% al 15% en la frontera norte.
Colombia añade un impuesto adicional del 8% para ciertos productos.
Argentina: el IVA ahora varía según si el cliente es responsable inscripto.

La abstracción genérica colapsó. La realidad de cada país es DISTINTA.
El "conocimiento" que parecía unificado (todos cobran IVA porcentual)
era una ilusión. Cada país tiene su propio motor fiscal.

Tuvimos que revertir: cada país, su propio calculador de impuestos.
Sí, hay código similar. Pero el CONOCIMIENTO es diferente.
Y DRY aplica a conocimiento, no a código.
```

### El Síndrome del "Utilero Loco"

```java
// ❌ El anti-patrón clásico: la clase Utils/BizCommon/Helpers
// que contiene TODO lo que "se repite" sin criterio de conocimiento

class UtilidadesGenerales {
    static String formatearFecha(Date fecha) { ... }
    static boolean validarEmail(String email) { ... }
    static String formatearMoneda(BigDecimal monto, String moneda) { ... }
    static String generarSlug(String texto) { ... }
    static boolean esMayorDeEdad(Date fechaNacimiento) { ... } // Dominio!
    static BigDecimal calcularDescuento(BigDecimal monto, String tipo) { ... } // Negocio!
    static String sanitizarHtml(String html) { ... }
    static String enmascararTarjeta(String numero) { ... }
    static boolean esHorarioLaboral(LocalDateTime momento) { ... } // Dominio!
    
    // 200 métodos más. Nadie sabe qué hace cada uno.
    // Cada PR toca esta clase. Merge conflicts constantes.
    // Imposible saber qué rompiste cuando cambias algo.
}
```

La clase `UtilidadesGenerales` (o `BizCommon`, `Helpers`, `Utils`) es el síntoma #1 de DRY mal aplicado. Es un cajón de sastre donde todo el conocimiento del sistema se pudre junto.

**La solución**: Agrupa por conocimiento, no por "son funciones sueltas".

```java
// ✅ Cada concepto en su lugar

// Conocimiento de presentación (formato, no reglas)
class FormateadorFechas { ... }
class FormateadorMoneda { ... }

// Conocimiento de dominio (reglas de negocio)
class VerificadorMayoriaEdad { ... }
class CalculadoraDescuento { ... }

// Conocimiento de seguridad
class EnmascaradorDatosSensibles { ... }
class SanitizadorHtml { ... }

// Conocimiento de infraestructura
class GeneradorSlug { ... }
```

---

## 5.2 KISS — Keep It Simple, Stupid (Mantenlo Simple, Imbécil)

### La Definición Incómoda

> "La simplicidad es la sofisticación suprema." — Leonardo da Vinci

KISS no es "hazlo simple porque eres tonto". Es "la complejidad se acumula sola; la simplicidad requiere esfuerzo consciente y disciplina".

### Por Qué lo Simple Es Tan Difícil

```
Hay dos tipos de complejidad en el software:

1. Complejidad ESENCIAL:
   Viene del problema que estás resolviendo. Es inevitable.
   Ej: "Necesitamos calcular impuestos en 5 países con regulaciones distintas."
   → No puedes simplificar esto. Es la naturaleza del problema.

2. Complejidad ACCIDENTAL:
   Viene de cómo resolviste el problema. Es evitable.
   Ej: "Usamos un framework de reglas de negocio que requiere 200 líneas
        de XML por cada país, más un motor de inferencia, más un DSL custom."
   → Esto SÍ puedes simplificarlo. Es complejidad que añadiste TÚ.
```

**El trabajo del arquitecto es minimizar la complejidad accidental.** La complejidad esencial ya es suficientemente dura.

### El Anti-Patrón del "Arquitecto Astronauta"

```
Situación real que presencié:

Necesidad: Un sistema para que los clientes VIP salten la cola de soporte.

Solución del arquitecto astronauta:
  • Motor de reglas de negocio (Drools) con DSL custom.
  • Apache Kafka para eventos de "cliente detectado".
  • Microservicio de Clasificación de Clientes.
  • Base de datos de grafos (Neo4j) para modelar relaciones VIP.
  • API Gateway con rate limiting diferenciado.
  • Dashboard en tiempo real con Apache Flink.
  Tiempo estimado: 4 meses. Equipo: 6 personas.

Solución del desarrollador sensato (que le ganó la discusión):
  • Un campo "vip" en la tabla de clientes.
  • Un IF en el sistema de tickets: if (cliente.vip) { prioridad = ALTA; }
  • Un filtro en la vista de soporte para ordenar por prioridad.
  Tiempo real: 3 días. Equipo: 1 persona.

Resultado: El cliente VIP recibió soporte prioritario en 3 días.
El arquitecto astronauta fue reasignado a "proyectos especiales".
```

### Código Simple vs Código Simplón

```python
# ❌ SIMPLÓN: Simple pero incorrecto, frágil, sin manejo de errores
def procesar_pedido(pedido_id):
    pedido = db.get_pedido(pedido_id)
    # ¿Y si el pedido no existe? ¿Si la BD falla?
    pago = stripe.cobrar(pedido.total)
    # ¿Y si Stripe está caído? ¿Si el pago es rechazado?
    db.marcar_pagado(pedido_id)
    # ¿Y si la BD falla aquí? ¿Perdemos el dinero?
    email.enviar(pedido.email, "¡Gracias por tu compra!")


# ✅ SIMPLE: Claro, maneja errores, decisiones explícitas, legible
def procesar_pedido(pedido_id):
    pedido = db.obtener_o_error(pedido_id)
    
    resultado = stripe.cobrar(pedido.total, pedido.metodo_pago)
    if resultado.rechazado:
        return ErrorPago(resultado.motivo)
    
    db.marcar_pagado(pedido_id, resultado.transaccion_id)
    email.enviar_confirmacion(pedido)
    return Exito()


# ❌ COMPLEJO INNECESARIAMENTE: Misma lógica, ilegible
def procesar_pedido(pedido_id):
    return (Pipeline()
        .add(ValidarPedido())
        .add(CalcularImpuestos())
        .add(ProcesarPago())
        .add(ActualizarInventario())
        .add(EnviarConfirmacion())
        .execute(pedido_id))
    # ¿Qué pasa si ValidarPedido falla? ¿Hace rollback?
    # ¿En qué orden se ejecutan? ¿Son síncronos o asíncronos?
    # Esto requiere leer la implementación de Pipeline, ValidarPedido,
    # CalcularImpuestos, ProcesarPago... para entender 10 líneas.
```

### La Prueba de la Simplicidad

Cuando escribas código, hazte estas preguntas:

```
1. ¿Puede un desarrollador nuevo en el equipo entender esto en 5 minutos?
2. ¿Hay alguna abstracción que puedo eliminar sin perder funcionalidad?
3. ¿Estoy resolviendo un problema que NO tengo hoy?
4. ¿Podría explicar esta solución a un Product Manager?
5. ¿Cuántas clases/interfaces/touchpoints tiene este cambio?
   Si la respuesta es >5 para algo conceptualmente simple, 
   probablemente es demasiado complejo.
```

---

## 5.3 YAGNI — You Ain't Gonna Need It (No Lo Vas a Necesitar)

### El Principio que Salva Startups (y Carreras)

> "Implementa cosas cuando realmente las necesites, no cuando preveas que las necesitarás."

YAGNI es el contrapeso de la sobre-ingeniería. Es el principio que te susurra al oído "no construyas un cohete para cruzar la calle".

### Lo que Cuesta Cada Línea de Código "Por Si Acaso"

```python
# "Voy a hacer esta función genérica por si acaso necesitamos más tipos"

# ❌ Hoy necesitas enviar un email. Solo un email.
def enviar_notificacion(
    destinatario: str,
    mensaje: str,
    tipo: str = "email",      # "Por si acaso SMS"
    prioridad: str = "normal", # "Por si acaso urgente"
    plantilla: str = None,     # "Por si acaso templates"
    adjuntos: list = None,     # "Por si acaso archivos"
    programado: datetime = None, # "Por si acaso envío diferido"
    idempotencia: str = None,  # "Por si acaso duplicados"
):
    if tipo == "email":
        # 5 líneas de lógica real
        pass
    # 30 líneas de parámetros que nadie usa hoy
    # 15 líneas de validación de combinaciones inválidas
    # 10 líneas de documentación de lo que "algún día podría ser"
```

**El costo invisible de YAGNI violado:**

```
Cada línea de código que escribes "por si acaso":
  • Debes mantenerla (cada refactor la toca).
  • Debes testearla (o tu cobertura baja).
  • Debes migrarla (cambios de versión, dependencias).
  • Debes documentarla (o el próximo dev pierde horas
    preguntándose "¿esto se usa?").
  • Puede contener bugs (y bugs en código no usado
    son los más difíciles de encontrar).
  • Añade carga cognitiva (el siguiente dev lee 50 líneas
    para entender lo que podrían ser 5).
  • Ralentiza el compilador, el linter, el IDE.
  • Ocupa espacio en la imagen Docker.
```

### El Caso Real que Cambió Mi Carrera

```
Año 2015. Startup. Sistema de reservas de hotel.

El CTO (Arquitecto):
  "Necesitamos una arquitectura de microservicios para escalar.
   Vamos a usar Kubernetes, Kafka, gRPC, y una malla de servicios.
   También necesitamos sharding de base de datos desde el día 1."

Yo (ingeniero senior):
  "Tenemos 200 usuarios. No 200 millones. 200."

El CTO:
  "Hay que pensar en grande. Netflix lo hace así."

Resultado después de 18 meses:
  • El sistema nunca pasó de 2,000 usuarios.
  • La startup gastó $15,000/mes en infraestructura para 2,000 usuarios.
  • Un monolito en un VPS de $40/mes hubiera sobrado.
  • El equipo de 12 personas pasaba 60% del tiempo lidiando con K8s,
    no construyendo features.
  • La startup cerró. No por falta de clientes. Por falta de dinero.
    El dinero se fue en infraestructura que nunca se necesitó.

Esto no es una historia. Es una autopsia. Estuve ahí. Lo vi morir.
```

### La Regla de YAGNI que Uso Todos los Días

```
"Si no tienes un requerimiento CONCRETO y una FECHA para necesitarlo,
 no lo construyas."

✅ "En Q3 entraremos a Brasil y necesitaremos multi-moneda" → OK, diseña
   para que sea fácil añadirlo, pero no lo implementes hasta Q3.

❌ "Algún día podríamos expandirnos internacionalmente" → YAGNI.
   Cuando llegue "algún día", tendrás más información, mejor contexto,
   y probablemente una solución más simple de la que imaginas hoy.
```

---

## 5.4 Separation of Concerns — Divide y Vencerás

### El Principio que Hace Posible Todo lo Demás

> "Divide tu sistema en secciones distintas, cada una abordando una preocupación separada."

Sin separación de concerns, no hay arquitectura posible. Es el principio más antiguo y más fundamental. Todo lo demás (SOLID, DDD, microservicios) son formas específicas de aplicar separación de concerns.

### Las Capas No Son el Enemigo — Las Capas Mal Hechas Sí

```java
// ❌ "Separación de concerns" mal hecha: capas que no separan NADA

// Capa Controller
@RestController
class PedidoController {
    @Autowired PedidoService pedidoService;
    
    @PostMapping("/pedidos")
    PedidoResponse crear(@RequestBody PedidoRequest req) {
        // ¿Validación aquí? ¿En el service? ¿En los dos?
        if (req.getItems().isEmpty()) {
            throw new ResponseStatusException(400, "Sin items");
        }
        // Formateo de datos de presentación en el controller?
        Pedido pedido = pedidoService.crearPedido(req);
        PedidoResponse resp = new PedidoResponse();
        resp.setId(pedido.getId().toString());
        resp.setTotal("$" + pedido.getTotal().toString());
        resp.setEstado(traducirEstado(pedido.getEstado()));
        return resp;
    }
}

// Capa Service
@Service
class PedidoService {
    @Autowired PedidoRepository repo;
    
    Pedido crearPedido(PedidoRequest req) {
        // Lógica de negocio... ¿o aquí?
        // Validación duplicada con el controller...
        // SQL injection? No, usamos JPA. ¿O sí?
        Pedido p = new Pedido();
        p.setClienteId(req.getClienteId());
        p.setItems(req.getItems());
        // Regla de negocio: máximo 50 items
        if (p.getItems().size() > 50) {
            throw new ReglaNegocioException("Máximo 50 items");
        }
        return repo.save(p);
    }
}
```

¿Dónde está la validación? ¿En el controller, en el service, en los dos? ¿Dónde está el formateo? ¿Qué pasa si el mismo caso de uso se llama desde la CLI, un job batch, o un mensaje de Kafka? El controller no existe en esos contextos. Si la validación está en el controller, los jobs batch no validan.

**El problema de fondo**: las "capas" no están separadas por concerns. Están separadas por nombres de paquetes. `controller`, `service`, `repository` no son concerns. Son naming.

### Separación Real de Concerns — por Responsabilidad, No por Capa

```java
// ✅ Cada concern en su lugar, independiente de cómo se invoque

// Concern: Validación de entrada (siempre se ejecuta, venga de donde venga)
class ValidarCreacionPedido {
    void validar(CrearPedidoCommand cmd) {
        if (cmd.getItems() == null || cmd.getItems().isEmpty()) {
            throw new ValidacionException("items", "El pedido debe tener al menos un item");
        }
        if (cmd.getItems().size() > 50) {
            throw new ValidacionException("items", "Máximo 50 items por pedido");
        }
        // Cada validación es explícita, testeable aisladamente
    }
}

// Concern: Caso de uso (orquestación pura, sin validación, sin formato)
class CrearPedidoUseCase {
    private final PedidoRepository repo;
    private final ValidarCreacionPedido validador;
    
    Pedido ejecutar(CrearPedidoCommand cmd) {
        validador.validar(cmd);
        Pedido pedido = Pedido.crear(cmd.getClienteId(), cmd.getItems());
        return repo.guardar(pedido);
    }
}

// Concern: Formato de respuesta HTTP (solo existe en el contexto REST)
@RestController
class PedidoRestController {
    private final CrearPedidoUseCase useCase;
    
    @PostMapping("/api/v1/pedidos")
    ResponseEntity<PedidoResponse> crear(@Valid @RequestBody CrearPedidoRequest req) {
        Pedido pedido = useCase.ejecutar(req.toCommand());
        return ResponseEntity.created(...).body(PedidoResponse.from(pedido));
    }
}

// Concern: Formato de respuesta CLI (diferente presentación, mismo caso de uso)
class PedidoCommandLine {
    private final CrearPedidoUseCase useCase;
    
    void ejecutar(String[] args) {
        CrearPedidoCommand cmd = parseArguments(args);
        try {
            Pedido pedido = useCase.ejecutar(cmd);
            System.out.println("✅ Pedido " + pedido.getId() + " creado. Total: " + pedido.getTotal());
        } catch (ValidacionException e) {
            System.err.println("❌ Error: " + e.getMessage());
            System.exit(1);
        }
    }
}

// Concern: Job batch (mismo caso de uso, sin UI)
class ProcesarPedidosBatchJob {
    private final CrearPedidoUseCase useCase;
    
    void procesarArchivo(File archivo) {
        leerCSV(archivo).forEach(cmd -> useCase.ejecutar(cmd));
    }
}
```

**La diferencia fundamental**: ahora `CrearPedidoUseCase` no sabe si fue invocado por REST, CLI, un batch o un mensaje de Kafka. La validación ocurre siempre. El formato de salida depende del adaptador. Eso es separación de concerns real.

---

## 5.5 Composition Over Inheritance — El Poder de Componer

### La Herencia No Es Tu Amiga

La herencia se enseña en la primera semana de programación orientada a objetos. Y es probablemente la herramienta más sobre-usada y peor utilizada de todo el paradigma.

```
¿Por qué duele tanto la herencia?

1. Es RÍGIDA: La jerarquía se define en tiempo de compilación.
   No puedes cambiar el comportamiento de un objeto en runtime.

2. Es FRÁGIL: Un cambio en la clase base puede romper TODAS las subclases.
   El "problema del cuadrado y el rectángulo" de LSP viene de la herencia.

3. Acopla PARA SIEMPRE: La subclase conoce (y depende de) los detalles
   internos de la clase base. Si la base tiene un bug, todas las subclases
   lo heredan. Si la base cambia, todas las subclases sufren.

4. Es LIMITADA: Solo puedes heredar de UNA clase (en Java, C#, etc.).
   La realidad no es así. Un objeto puede ser muchas cosas.
```

### El Sistema de Personajes que Enseña Por Qué Composición > Herencia

```java
// ❌ HERENCIA: Un infierno de jerarquías

class Personaje {
    int vida;
    int x, y;
    void mover(int dx, int dy) { this.x += dx; this.y += dy; }
    void recibirDaño(int daño) { this.vida -= daño; }
}

class PersonajeVolador extends Personaje {
    void volar(int dx, int dy, int dz) { /* ... */ }
}

class PersonajeNadador extends Personaje {
    void nadar(int dx, int dy) { /* ... */ }
}

// Hasta aquí bien. Pero el juego crece:

// Necesito un personaje que vuele Y nade.
// ¿Extiendo PersonajeVolador o PersonajeNadador?
// ¿Creo PersonajeVoladorNadador? ¿Y si también escala paredes?

class Dragon extends PersonajeVolador { // ¿Y nadar?
    // El dragón nada en el juego original. No puedo modelarlo.

// Necesito un personaje que lance hechizos y vuele.
// Necesito un personaje que lance hechizos, vuele y nade.
// Necesito un personaje que escale paredes y sea invisible.
// Con herencia: 2^n combinaciones = explosión combinatoria.
```

```java
// ✅ COMPOSICIÓN: Flexibilidad total

// Las capacidades son COMPONENTES, no clases base
interface CapacidadMovimiento {
    void mover(Personaje personaje, int dx, int dy);
}

class Caminar implements CapacidadMovimiento {
    public void mover(Personaje p, int dx, int dy) {
        p.setX(p.getX() + dx);
        p.setY(p.getY() + dy);
    }
}

class Volar implements CapacidadMovimiento {
    public void mover(Personaje p, int dx, int dy) {
        p.setX(p.getX() + dx);
        p.setY(p.getY() + dy);
        p.setAltura(p.getAltura() + 1); // Puede ignorar terreno
    }
}

class Nadar implements CapacidadMovimiento {
    public void mover(Personaje p, int dx, int dy) {
        // Mitad de velocidad en agua, puede cruzar ríos
        p.setX(p.getX() + dx / 2);
        p.setY(p.getY() + dy / 2);
    }
}

class Teletransportarse implements CapacidadMovimiento {
    public void mover(Personaje p, int dx, int dy) {
        // Sin colisiones, instantáneo
        p.setX(dx);
        p.setY(dy);
    }
}

interface CapacidadCombate {
    int calcularDaño(Personaje atacante, Personaje defensor);
}

class AtaqueEspada implements CapacidadCombate {
    public int calcularDaño(Personaje atacante, Personaje defensor) {
        return atacante.getFuerza() * 2;
    }
}

class LanzarHechizo implements CapacidadCombate {
    public int calcularDaño(Personaje atacante, Personaje defensor) {
        return atacante.getInteligencia() * 3 - defensor.getResistenciaMagica();
    }
}

// Ahora cada personaje COMPONE sus capacidades
class Personaje {
    private String nombre;
    private int vida, fuerza, inteligencia, x, y;
    
    // Composición: un personaje PUEDE tener múltiples capacidades
    private List<CapacidadMovimiento> movimientos = new ArrayList<>();
    private CapacidadCombate ataque;
    
    void añadirMovimiento(CapacidadMovimiento m) { movimientos.add(m); }
    void setAtaque(CapacidadCombate a) { this.ataque = a; }
    
    void mover(int dx, int dy) {
        for (CapacidadMovimiento m : movimientos) {
            m.mover(this, dx, dy);
        }
    }
    
    int atacar(Personaje defensor) {
        return ataque.calcularDaño(this, defensor);
    }
}

// Ahora creo CUALQUIER personaje sin límites:

// Un dragón que vuela, nada Y lanza fuego
Personaje dragon = new Personaje("Dragón", 500, 50, 30);
dragon.añadirMovimiento(new Volar());
dragon.añadirMovimiento(new Nadar());
dragon.setAtaque(new LanzarHechizo()); // Aliento de fuego = hechizo

// Un ninja que camina, escala y ataca con espada
Personaje ninja = new Personaje("Ninja", 100, 40, 10);
ninja.añadirMovimiento(new Caminar());
ninja.añadirMovimiento(new EscalarParedes()); // Fácil de añadir
ninja.setAtaque(new AtaqueEspada());

// Un fantasma que solo se teletransporta (sin combate)
Personaje fantasma = new Personaje("Fantasma", 50, 0, 0);
fantasma.añadirMovimiento(new Teletransportarse());
// No tiene ataque. No puede combatir. Es correcto.

// AÑADIR NUEVA CAPACIDAD = clase nueva. Cero impacto en lo existente.
class EscalarParedes implements CapacidadMovimiento {
    public void mover(Personaje p, int dx, int dy) {
        if (dy > 0) { // Subiendo: más lento
            p.setY(p.getY() + dy / 2);
        } else {
            p.setY(p.getY() + dy);
        }
    }
}
```

**La diferencia entre herencia y composición es la diferencia entre un tren (herencia: rígido, sigue sus rieles) y un auto (composición: flexible, puedes cambiarle las ruedas, el motor y la dirección independientemente).**

---

## 5.6 Fail Fast — Falla Rápido, Falla Fuerte, Falla Temprano

### El Principio que Te Ahorra Madrugadas de Debugging

> "Detecta errores lo antes posible en el ciclo de vida de la aplicación."

El momento más barato para encontrar un bug es durante la compilación. El segundo más barato es durante los tests. El más caro es en producción a las 3 AM con el CEO llamándote.

### El Costo de Fallar Tarde

```java
// ❌ FALLAR TARDE — El error se manifiesta muy lejos de su causa

// Archivo 1: ClienteService.java
public class ClienteService {
    public void actualizarEmail(String clienteId, String email) {
        Cliente cliente = repo.findById(clienteId);
        cliente.setEmail(email); // ¿Es válido el email? No validamos.
        repo.save(cliente);
        // Aquí todo parece bien...
    }
}

// Archivo 2: EmailService.java (EJECUTADO 5 HORAS DESPUÉS)
public class EmailService {
    public void enviarBoletinSemanal() {
        List<Cliente> clientes = repo.findAll();
        for (Cliente c : clientes) {
            emailSender.send(c.getEmail(), "Boletín Semanal", contenido);
            // ¿Qué pasa si un email es null? ¿O "juan@"?
        }
    }
}

// Archivo 3: Tu teléfono a las 3 AM
// "El envío del boletín falló. 50,000 clientes no lo recibieron.
//  El error es NullPointerException en EmailSender.send().
//  No sabemos qué cliente tiene el email roto.
//  El cliente se dio de alta hace 5 horas. No sabemos quién fue."
```

```java
// ✅ FALLAR RÁPIDO — El error se detecta en el MOMENTO de la causa

public class ClienteService {
    public void actualizarEmail(String clienteId, String email) {
        // Validación INMEDIATA. Si el email es inválido, explota AQUÍ.
        requireNonNull(email, "El email no puede ser null");
        requireValidEmail(email, "Formato de email inválido");
        
        Cliente cliente = repo.findById(clienteId);
        requireNonNull(cliente, "Cliente " + clienteId + " no existe");
        
        cliente.setEmail(email);
        repo.save(cliente);
        // Si llegó aquí, el email ES VÁLIDO. Garantizado.
    }
}

// EmailService: NUNCA recibirá un email inválido.
// Si llega aquí, los datos son correctos. Garantía del sistema.
public class EmailService {
    public void enviarBoletinSemanal() {
        List<Cliente> clientes = repo.findAll();
        for (Cliente c : clientes) {
            emailSender.send(c.getEmail(), "Boletín Semanal", contenido);
        }
        // Si esto falla, es culpa del servidor SMTP, no de nuestros datos.
    }
}
```

### Las 3 Capas del Fail Fast

```
1. Fail Fast en COMPILACIÓN:
   Usa tipos específicos, no Strings genéricos.
   
   ❌ void actualizarEstado(String pedidoId, String estado)
   ✅ void actualizarEstado(PedidoId pedidoId, EstadoPedido estado)
   
   Si EstadoPedido es un enum, no puedes pasar "PENDIENT" (typo).
   El compilador te salva. Zero bugs en producción por typos.

2. Fail Fast en RUNTIME TEMPRANO:
   Valida al inicio del método, no al final.
   
   ❌ Procesar todo y validar al guardar.
   ✅ Validar TODO en las primeras 3 líneas. Si algo falla,
      no gastaste CPU, BD, red en un request que iba a fallar.

3. Fail Fast en los CONTRATOS:
   Documenta lo que esperas y lo que garantizas.
   
   ❌ "Este método devuelve un Pedido."
   ✅ "Este método devuelve un Pedido NUNCA NULO. Si el ID no existe,
      lanza PedidoNotFoundException. El pedido siempre tiene al menos
      un item. El total nunca es negativo."
```

---

## 5.7 Convention Over Configuration — Convención Sobre Configuración

### El Principio que Hace que un Equipo de 50 Trabaje como Uno Solo

> "Establece convenciones razonables. Solo configura lo que se desvía de la convención."

Cuando un desarrollador nuevo abre tu proyecto, debería poder adivinar dónde están las cosas. No debería necesitar leer 200 páginas de documentación para encontrar un controlador REST.

### Sin Convención: El Infierno de las 50 Estructuras de Proyecto

```
Equipo A:
  src/main/java/com/empresa/pedidos/
    controllers/PedidoController.java
    services/PedidoService.java
    repositories/PedidoRepo.java

Equipo B:
  src/main/java/com/empresa/usuarios/
    web/UsuarioController.java
    core/UsuarioService.java
    data/UsuarioDao.java

Equipo C:
  src/main/java/com/empresa/productos/
    adapter/rest/ProductoRestAdapter.java
    domain/Producto.java
    application/ProductoUseCase.java
    infrastructure/postgres/PostgresProductoRepo.java

Un dev pasa del Equipo A al Equipo B:
  "¿Dónde está el controlador?"
  "En web/, no en controllers/."
  "¿Y el repositorio?"
  "Se llama Dao, no Repository."
  
  Tiempo perdido: horas cada semana. Estrés acumulado.
```

**Con convención (ejemplo: Clean Architecture estandarizada):**

```
Equipo A (pedidos):
  pedidos/
    domain/    → Pedido.java, PedidoRepository.java (interfaz)
    application/ → CrearPedidoUseCase.java
    infrastructure/ → PostgresPedidoRepository.java
    adapter/rest/ → PedidoRestController.java

Equipo B (usuarios):
  usuarios/
    domain/    → Usuario.java, UsuarioRepository.java (interfaz)
    application/ → RegistrarUsuarioUseCase.java
    infrastructure/ → PostgresUsuarioRepository.java
    adapter/rest/ → UsuarioRestController.java

Equipo C (productos):
  productos/
    domain/    → Producto.java, ProductoRepository.java (interfaz)
    application/ → BuscarProductoUseCase.java
    infrastructure/ → PostgresProductoRepository.java
    adapter/rest/ → ProductoRestController.java

Cualquier dev sabe dónde está cada cosa en CUALQUIER equipo.
El onboarding baja de 4 meses a 2 semanas.
```

---

## 5.8 Principio de Menor Asombro (POLA)

> "El comportamiento de un componente debe ser el que la mayoría de usuarios esperarían."

Si tu función `getUser(id)` envía un email, actualiza un contador en Redis y modifica el avatar del usuario... estás violando POLA. Eres un psicópata del código y tus compañeros te odian en silencio.

### Ejemplo de Violación de POLA que Vi en Producción

```java
// Un getter... ¿qué podría salir mal?

class Pedido {
    private List<Item> items;
    private BigDecimal totalCache;
    
    // ❌ PSICÓPATA: Este getter recalcula el total CADA VEZ
    List<Item> getItems() {
        // "Optimización": recargar de BD si no están en caché
        if (items == null) {
            items = database.loadItems(this.id); // LLAMADA A BD EN UN GETTER
        }
        return items;
    }
    
    BigDecimal getTotal() {
        // "Por si acaso el total no está actualizado, recalculamos"
        if (totalCache == null || itemsHanCambiado()) {
            totalCache = recalcularTotal(); // Recalcula CADA VEZ
        }
        return totalCache;
    }
    
    // Resultado: código inocente que hace 3 getters = 3 queries a BD
    // log.info("Total del pedido: {}", pedido.getTotal()); // Query a BD en un log
}
```

**La regla**: si tu método se llama `get`, solo debe devolver un valor. Si se llama `find`, puede buscar. Si se llama `calculate`, puede calcular. El nombre debe describir el comportamiento. Cada vez que violas esto, alguien pierde 4 horas debuggeando un problema de rendimiento "inexplicable".

---

## 5.9 Postel's Law — El Principio de Robustez

> "Sé conservador en lo que envías, liberal en lo que aceptas."

Tu código debe producir output estricto y predecible, pero aceptar input flexible y tolerante. Es el principio detrás de que Internet funcione: un servidor web acepta headers en cualquier orden y con variaciones de formato, pero responde con headers perfectamente formados.

```python
# ✅ Aplicando Postel's Law a una API

# INPUT: Liberal (aceptamos variaciones que son razonables)
def parse_fecha(valor):
    """Acepta múltiples formatos de fecha, pero devuelve uno estricto."""
    formatos_permitidos = [
        "%Y-%m-%d",           # 2024-01-15
        "%Y-%m-%dT%H:%M:%SZ", # 2024-01-15T10:30:00Z
        "%d/%m/%Y",           # 15/01/2024 (formato LATAM)
        "%m/%d/%Y",           # 01/15/2024 (formato US)
    ]
    for fmt in formatos_permitidos:
        try:
            return datetime.strptime(valor, fmt)
        except ValueError:
            continue
    raise ValueError(f"Formato de fecha no reconocido: {valor}")

# OUTPUT: Conservador (siempre devolvemos el mismo formato)
def formatear_fecha(fecha: datetime) -> str:
    return fecha.strftime("%Y-%m-%dT%H:%M:%SZ")  # ISO 8601, siempre
```

---

## 5.10 Ejercicio Final — La Prueba de Fuego

Te entregan este código legacy. Tienes 4 horas para hacerlo mantenible. ¿Qué principios aplicas y cómo?

```python
# Sistema de descuentos para e-commerce
# 800 líneas. Nadie sabe cómo funciona. El autor original renunció.

class DiscountEngine:
    def calculate(self, order, customer, date, promo_code=None):
        total = 0
        for item in order.items:
            total += item.price * item.quantity
        
        # Descuento por categoría de cliente
        if customer.type == "VIP":
            total *= 0.8
        elif customer.type == "PREMIUM":
            total *= 0.9
        elif customer.type == "REGULAR":
            total *= 0.95
        
        # Descuento estacional
        if date.month == 12 and date.day >= 20:  # Navidad
            total *= 0.85
        elif date.month == 11 and date.day >= 15 and date.day <= 30:  # Black Friday
            total *= 0.7
        elif date.month == 7 and date.day >= 1 and date.day <= 31:  # Julio
            total *= 0.9
        
        # Descuento por volumen
        item_count = len(order.items)
        if item_count > 10:
            total *= 0.9
        elif item_count > 5:
            total *= 0.95
        
        # Código promocional
        if promo_code:
            if promo_code == "BIENVENIDA10":
                total *= 0.9
            elif promo_code == "VERANO2024":
                if date.month in [6, 7, 8]:
                    total *= 0.85
            elif promo_code == "VIP50" and customer.type == "VIP":
                total *= 0.5
        
        # El total nunca puede ser menor que 0
        return max(total, 0)
```

Tu tarea: refactoriza esto aplicando DRY, KISS, YAGNI, SoC, Composition, Fail Fast. Explica qué principio usaste en cada cambio y por qué.

---

> **Reflexión del capítulo**: Los principios de diseño no se aprenden en un libro. Se aprenden sufriendo las consecuencias de no aplicarlos. Cada vez que violas DRY y duplicas conocimiento, alguien va a tener que arreglar 47 archivos en un fin de semana. Cada vez que violas KISS y sobre-ingenierizas, alguien va a maldecir tu nombre mientras debuggea 50 capas de abstracción para algo que debían ser 20 líneas. Sé el arquitecto cuyo código otros quieren heredar, no el que todos temen tocar.
