# Capítulo 7: Arquitectura Hexagonal y Clean Architecture

> "El dominio es el rey. La infraestructura es el sirviente." — Alistair Cockburn

## 7.1 El Problema: Acoplamiento a Frameworks

La mayoría de aplicaciones se escriben al revés: primero elegimos Spring Boot, Express o Django, y luego metemos la lógica de negocio como podemos.

**El resultado**: código de negocio mezclado con controladores HTTP, anotaciones de ORM y configuraciones de framework. Cambiar cualquier pieza externa implica reescribir el núcleo.

## 7.2 Arquitectura Hexagonal (Ports & Adapters)

Propuesta por Alistair Cockburn en 2005. La metáfora: un hexágono con el dominio en el centro, y adaptadores enchufables en cada lado.

```
        ┌──────────────────────────────────┐
        │      Adaptadores Primarios        │
        │   (REST, CLI, GUI, Test, gRPC)    │
        │              │                    │
        │         ┌────▼─────┐              │
        │         │  PUERTOS  │              │
        │      ┌──┤  (API)   ├──┐           │
        │      │  └──────────┘  │           │
        │  ┌───▼───┐       ┌───▼───┐       │
        │  │ Casos │       │ Casos │       │
        │  │  Uso  │◄─────►│  Uso  │       │
        │  └───┬───┘       └───┬───┘       │
        │      │               │           │
        │  ┌───▼───────────────▼───┐       │
        │  │      DOMINIO           │       │
        │  │  Entidades, VOs,       │       │
        │  │  Servicios de dominio  │       │
        │  └───┬───────────────┬───┘       │
        │      │               │           │
        │  ┌───▼───┐       ┌───▼───┐       │
        │  │PUERTOS│       │PUERTOS│       │
        │  │ (SPI) │       │ (SPI) │       │
        │  └───┬───┘       └───┬───┘       │
        │      │               │           │
        │  ┌───▼───┐       ┌───▼───┐       │
        │  │  BD   │       │ Email │       │
        │  │Adapter│       │Adapter│       │
        │  └───────┘       └───────┘       │
        │      Adaptadores Secundarios      │
        └──────────────────────────────────┘
```

### Conceptos Clave

**Puertos (Ports)**: Interfaces que definen qué necesita o provee el dominio.
- **Puertos primarios (API)**: Lo que el dominio ofrece al exterior (casos de uso).
- **Puertos secundarios (SPI)**: Lo que el dominio necesita del exterior (repositorios, servicios).

**Adaptadores (Adapters)**: Implementaciones concretas que conectan puertos con tecnología real.
- **Primarios**: Controladores REST, handlers gRPC, CLI commands.
- **Secundarios**: DAOs de PostgreSQL, clientes HTTP, adaptadores de AWS S3.

### La Regla de Dependencia

> **El código fuente solo apunta hacia adentro. El dominio no conoce nada externo.**

```
Dirección de las dependencias:
Framework ──► Adaptadores ──► Aplicación ──► Dominio
                                         (NO al revés)
```

## 7.3 Clean Architecture (Robert C. Martin)

Esencialmente la misma idea que Hexagonal, con capas explícitas:

```
┌──────────────────────────────────────────────┐
│  Frameworks & Drivers (Web, DB, UI, Devices) │  ← Capa exterior
├──────────────────────────────────────────────┤
│  Interface Adapters (Controllers, Presenters)│
├──────────────────────────────────────────────┤
│  Application Use Cases (Casos de uso)        │
├──────────────────────────────────────────────┤
│  Enterprise Business Rules (Entidades)       │  ← Capa interior
└──────────────────────────────────────────────┘
```

### Estructura de Paquetes Típica

```
com.empresa.proyecto/
├── domain/
│   ├── model/          # Entidades, Value Objects
│   ├── service/        # Servicios de dominio
│   └── port/           # Interfaces de puertos (repositorios, etc.)
├── application/
│   ├── usecase/        # Casos de uso (orquestación)
│   └── port/           # Interfaces de puertos de entrada
├── infrastructure/
│   ├── persistence/    # Implementaciones JPA/Mongo/etc.
│   ├── messaging/      # Kafka, RabbitMQ
│   └── external/       # Clientes de APIs externas
└── adapter/
    ├── rest/           # Controladores REST
    ├── grpc/           # Handlers gRPC
    └── cli/            # Comandos de terminal
```

## 7.4 Ejemplo Práctico

```java
// ─── DOMINIO (no depende de nada) ───
public class Pedido {
    private PedidoId id;
    private EstadoPedido estado;

    public void confirmar() {
        if (estado != EstadoPedido.PENDIENTE) {
            throw new PedidoNoConfirmableException();
        }
        this.estado = EstadoPedido.CONFIRMADO;
        // Domain event
    }
}

// ─── PUERTO (interfaz en dominio) ───
public interface PedidoRepository {
    Optional<Pedido> findById(PedidoId id);
    void save(Pedido pedido);
}

// ─── CASO DE USO (aplicación) ───
public class ConfirmarPedidoUseCase {
    private final PedidoRepository repo;

    public void ejecutar(PedidoId id) {
        Pedido pedido = repo.findById(id)
            .orElseThrow(() -> new PedidoNoEncontradoException(id));
        pedido.confirmar();
        repo.save(pedido);
    }
}

// ─── ADAPTADOR (infraestructura, capa externa) ───
@RestController
public class PedidoController {
    private final ConfirmarPedidoUseCase useCase;

    @PostMapping("/pedidos/{id}/confirmar")
    public ResponseEntity<Void> confirmar(@PathVariable String id) {
        useCase.ejecutar(new PedidoId(id));
        return ResponseEntity.ok().build();
    }
}
```

## 7.5 Beneficios Reales

| Beneficio | Cómo se Logra |
|-----------|--------------|
| **Testabilidad** | Dominio sin dependencias externas = tests unitarios rápidos. |
| **Reemplazabilidad** | Cambiar PostgreSQL por MongoDB = un nuevo adaptador. |
| **Evolución Independiente** | Cambiar REST por gRPC no toca el dominio. |
| **Enfoque en Negocio** | El código refleja el problema, no la tecnología. |
| **Onboarding** | Nuevos devs entienden el negocio leyendo el dominio. |

## 7.6 Cuándo NO usar Clean/Hexagonal

- **CRUD simple** sin lógica de negocio real.
- **Prototipos/MVPs** donde el time-to-market es crítico.
- **Serverless Functions** simples (un handler con 50 líneas no necesita 4 capas).
- **Equipos muy pequeños** que no justifican la sobrecarga estructural.

---

> **Reflexión del capítulo**: La arquitectura hexagonal y Clean Architecture no son varitas mágicas. Son disciplinas. El verdadero valor no está en el diagrama bonito, sino en la disciplina diaria de mantener las dependencias apuntando hacia adentro.

---

← [Capítulo anterior](06-ddd.md) | [Inicio](README.md) | [Capítulo siguiente →](08-estilos-arquitectonicos.md)
