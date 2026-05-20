# Capítulo 6: Domain-Driven Design (DDD)

> "El software debe modelar el dominio del problema, no la tecnología que lo implementa." — Eric Evans

## 6.1 ¿Qué es DDD?

Domain-Driven Design es un enfoque de diseño de software que coloca el **dominio del negocio** en el centro de todo. No es un patrón, es una filosofía de diseño.

**Idea central**: El código debe hablar el lenguaje del negocio. Si un experto del dominio no entiende tu código, tienes un problema.

## 6.2 Conceptos Fundamentales

### Ubiquitous Language (Lenguaje Ubicuo)
Un lenguaje común entre desarrolladores y expertos del dominio. Las mismas palabras significan lo mismo en conversaciones, código, documentación y tests.

```
Ejemplo en e-commerce:
- "Pedido" (no "OrderRecord" o "OrderDTO")
- "confirmar pedido" (no "updateOrderStatus(5)")
- "artículo agotado" (no "throw InventoryException(code=402)")
```

### Bounded Contexts (Contextos Delimitados)
Cada contexto tiene su propio modelo, su propio ubiquitous language, y sus propias reglas.

```
┌──────────────────────────────────────────────┐
│        Sistema E-commerce                     │
│                                               │
│  ┌──────────────┐  ┌──────────────┐          │
│  │   Ventas     │  │   Envíos     │          │
│  │              │  │              │          │
│  │  "Producto"  │  │  "Producto"  │          │
│  │  (precio,    │  │  (peso,      │          │
│  │   descuento) │  │   dimensiones│          │
│  │              │  │   fragilidad) │          │
│  └──────────────┘  └──────────────┘          │
│                                               │
│  "Producto" significa cosas diferentes        │
│  en cada contexto, ¡y está bien!              │
└──────────────────────────────────────────────┘
```

## 6.3 Building Blocks Tácticos

### Entities
Objetos con identidad continua a través del tiempo. Dos entidades son diferentes aunque todos sus atributos sean iguales.

```java
// Identity es lo que importa
class Cliente {
    private ClienteId id;      // La identidad
    private String nombre;      // Puede cambiar
    private String email;       // Puede cambiar
    // Dos clientes con id distinto son entidades diferentes
}
```

### Value Objects
Objetos definidos por sus atributos, sin identidad. Son inmutables y reemplazables.

```java
// Definido por sus valores, no por identidad
class Dinero {
    private BigDecimal cantidad;
    private Moneda moneda;

    // Dos objetos con misma cantidad y moneda son iguales
    // Son inmutables: sumar() devuelve un nuevo Dinero
}
```

### Aggregates
Un cluster de entidades y value objects tratados como una unidad. Tiene una **raíz** que es el único punto de acceso.

```java
// Pedido es la raíz del agregado
class Pedido {  // Aggregate Root
    private PedidoId id;
    private List<LineaPedido> lineas;  // Solo accesible vía Pedido
    private DireccionEnvio direccion;   // Value Object

    void añadirLinea(Producto p, int cantidad) {
        // Invariante: no más de 50 líneas
        if (lineas.size() >= 50) throw new DemasiadasLineasException();
        lineas.add(new LineaPedido(p, cantidad));
    }
}
```

### Repositories
Abstracción para acceder a aggregates. Parecen colecciones en memoria.

```java
interface PedidoRepository {
    Optional<Pedido> findById(PedidoId id);
    void save(Pedido pedido);
    List<Pedido> findByClienteId(ClienteId clienteId);
}
```

### Domain Services
Lógica que no pertenece naturalmente a una entidad o value object.

```java
// Transferencia entre cuentas: ¿dónde va la lógica?
class ServicioTransferencia {
    void transferir(Cuenta origen, Cuenta destino, Dinero monto) {
        origen.retirar(monto);
        destino.depositar(monto);
    }
}
```

### Domain Events
Hechos importantes que ocurren en el dominio.

```java
// Algo relevante pasó
record PedidoConfirmado(PedidoId pedidoId, ClienteId clienteId, Instant cuando) { }
record ProductoAgotado(ProductoId productoId) { }
```

## 6.4 Strategic Design

### Context Mapping
Define las relaciones entre bounded contexts:

| Relación | Descripción |
|----------|-------------|
| **Partnership** | Dos equipos colaboran estrechamente. |
| **Shared Kernel** | Comparten un subconjunto del modelo. |
| **Customer-Supplier** | Upstream provee, downstream consume. |
| **Conformist** | Downstream se adapta sin influir en upstream. |
| **Anticorruption Layer (ACL)** | Traduce entre modelos para evitar contaminación. |
| **Open Host Service** | API bien definida para múltiples consumidores. |
| **Published Language** | Formato documentado (XML, JSON Schema, Protobuf). |

### Anti-Corruption Layer (ACL)
El patrón más importante. Aísla tu dominio del modelo de sistemas externos.

```
[Tu Contexto] <──> [ACL] <──> [Sistema Legacy Externo]
                           (traduce terminología,
                            adapta formatos,
                            filtra ruido)
```

## 6.5 Event Storming

Técnica colaborativa para descubrir el dominio:
1. **Eventos de dominio** (naranja) — ¿Qué ocurre?
2. **Comandos** (azul) — ¿Qué dispara el evento?
3. **Aggregates** (amarillo) — ¿Qué entidad maneja el comando?
4. **Políticas** (lila) — ¿Qué proceso reacciona a eventos?
5. **Read Models** (verde) — ¿Qué datos necesita el usuario?
6. **Sistemas externos** (rosa) — ¿Qué hay fuera del sistema?

## 6.6 Cuándo Usar DDD

**Usa DDD cuando:**
- El dominio es complejo (no CRUD simple).
- Hay reglas de negocio que cambian frecuentemente.
- Múltiples stakeholders con distintos lenguajes.

**No uses DDD cuando:**
- Es un CRUD simple (la complejidad añadida no compensa).
- El dominio es puramente técnico (ej: un API Gateway).
- El equipo no tiene acceso a expertos del dominio.

---

> **Reflexión del capítulo**: DDD no es sobre tecnología, es sobre entender el negocio. El mejor código del mundo es inútil si resuelve el problema equivocado. Pasa tiempo con los que conocen el dominio.
