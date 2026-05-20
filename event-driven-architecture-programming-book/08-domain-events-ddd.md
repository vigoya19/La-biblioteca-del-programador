# Capítulo 8: Domain Events y Domain-Driven Design

Domain-Driven Design (DDD) y Event-Driven Architecture (EDA) son dos paradigmas que se potencian mutuamente. Los **Domain Events** son el pegamento: representan cambios significativos en el dominio del negocio y permiten que los Bounded Contexts colaboren sin acoplarse.

> "Un Domain Event es algo que los expertos del negocio mencionan en las conversaciones: 'cuando se crea una orden', 'cuando se procesa un pago', 'cuando se envía un producto'." — Eric Evans

---

## 8.1 Domain Events como Ciudadanos de Primera Clase

En DDD tradicional, las entidades y value objects son los protagonistas. En EDA + DDD, los **eventos** también son parte del modelo de dominio:

```typescript
// ─── Domain Event: parte del lenguaje ubicuo ───
namespace Ordenes {
  export interface OrdenCreada {
    type: "orden.creada";
    data: {
      ordenId: string;
      clienteId: string;
      items: { productoId: string; cantidad: number; precioUnitario: number }[];
      total: number;
      direccionEnvio: Direccion;
    };
    metadata: EventMetadata;
  }

  export interface OrdenPagada {
    type: "orden.pagada";
    data: {
      ordenId: string;
      metodoPago: "tarjeta" | "transferencia" | "paypal";
      transaccionId: string;
      monto: number;
    };
    metadata: EventMetadata;
  }

  export interface OrdenEnviada {
    type: "orden.enviada";
    data: {
      ordenId: string;
      transportista: string;
      trackingNumber: string;
      fechaEnvio: string;
    };
    metadata: EventMetadata;
  }
}

// Los Domain Events deben ser nombrados en lenguaje de negocio:
// ✅ "orden.pagada", "producto.reservado", "cliente.registrado"
// ❌ "orden.actualizada", "datos.guardados", "registro.modificado"
```

### Eventos dentro del Aggregate

Los aggregates emiten eventos como resultado de comandos exitosos:

```typescript
class Orden {
  private ordenId: string;
  private estado: OrdenEstado;
  private items: ItemOrden[];
  private total: number;
  private eventos: DomainEvent[] = []; // Eventos pendientes de publicar

  private constructor(ordenId: string) {
    this.ordenId = ordenId;
    this.estado = "creada";
    this.items = [];
    this.total = 0;
    this.eventos = [];
  }

  // ─── Método factory: crea y emite OrdenCreada ───
  static crear(id: string, clienteId: string, items: ItemDTO[]): Orden {
    const orden = new Orden(id);

    const itemsOrden = items.map(i => new ItemOrden(i.productoId, i.cantidad, i.precioUnitario));
    orden.items = itemsOrden;
    orden.total = itemsOrden.reduce((sum, i) => sum + i.subtotal(), 0);

    // Emitir Domain Event
    orden.eventos.push({
      type: "orden.creada",
      data: {
        ordenId: id,
        clienteId,
        items: itemsOrden.map(i => i.toDTO()),
        total: orden.total,
        direccionEnvio: items[0].direccionEnvio, // Simplificado
      },
      metadata: {
        eventId: crypto.randomUUID(),
        timestamp: new Date().toISOString(),
        aggregateType: "Orden",
        aggregateId: id,
        version: 1,
      },
    });

    return orden;
  }

  // ─── Comando: pagar ───
  pagar(metodo: MetodoPago, transaccionId: string): void {
    if (this.estado !== "creada") {
      throw new Error(`No se puede pagar una orden en estado ${this.estado}`);
    }

    this.estado = "pagada";

    this.eventos.push({
      type: "orden.pagada",
      data: {
        ordenId: this.ordenId,
        metodoPago: metodo,
        transaccionId,
        monto: this.total,
      },
      metadata: {
        eventId: crypto.randomUUID(),
        timestamp: new Date().toISOString(),
        aggregateType: "Orden",
        aggregateId: this.ordenId,
        version: 2,
      },
    });
  }

  // ─── Extraer eventos para publicar ───
  drenarEventos(): DomainEvent[] {
    const eventos = [...this.eventos];
    this.eventos = [];
    return eventos;
  }
}

// ─── Uso en el Application Service ───
async function crearOrdenHandler(cmd: CrearOrdenComando): Promise<string> {
  const orden = Orden.crear(cmd.ordenId, cmd.clienteId, cmd.items);
  await repositorio.guardar(orden);

  // Publicar eventos generados por el aggregate
  const eventos = orden.drenarEventos();
  for (const evento of eventos) {
    await eventBus.publicar(evento);
  }

  return cmd.ordenId;
}
```

---

## 8.2 Event Storming: Descubrir Eventos del Negocio

Event Storming es una técnica de workshop creada por Alberto Brandolini para modelar sistemas complejos usando eventos como protagonistas:

```
Event Storming: secuencia temporal de Domain Events

   Cliente        Productos        Cliente        Orden         Producto
  Registrado      Agregados al    Realiza Pago    Pagada        Reservado
      │             Carrito           │              │              │
      ▼                ▼              ▼              ▼              ▼
  ─────●──────────────●─────────────●─────────────●─────────────●──────▶ tiempo
                                                        │
                                                    ┌───▼────┐
                                                    │ Orden  │
                                                    │ Enviada│
                                                    └────────┘
```

### Fases del Event Storming

```
Fase 1: Brainstorming (orange stickies)
  "¿Qué ocurre en el negocio?"
  → ClienteRegistrado, ProductoAgregado, OrdenCreada, PagoRealizado...

Fase 2: Timeline (ordenar cronológicamente)
  → Secuencia correcta de eventos

Fase 3: Hotspots (preguntas sin responder)
  → "¿Qué pasa si el pago falla?" → purple stickies

Fase 4: Commands (acciones que disparan eventos)
  → RegistrarCliente → ClienteRegistrado
  → CrearOrden → OrdenCreada
  → ProcesarPago → PagoProcesado

Fase 5: Aggregates (agrupar eventos relacionados)
  → Cliente, Orden, Pago, Envío

Fase 6: Bounded Contexts (fronteras del dominio)
  → Contexto de Órdenes, Contexto de Pagos, Contexto de Envíos

Fase 7: Policies (reacciones automáticas)
  → "Cuando OrdenPagada → Iniciar Envío"
```

```typescript
// Resultado del Event Storming: código que refleja el modelo descubierto

// Policy: reacción automática a un evento
// "Cuando una orden es pagada, iniciar el proceso de envío"
const cuandoOrdenPagadaIniciarEnvio = async (evento: OrdenPagada) => {
  const orden = await repositorioOrdenes.obtener(evento.data.ordenId);
  const guia = await servicioEnvios.crearGuia(orden);
  // Emitir nuevo evento
  await eventBus.publicar({
    type: "envio.guia_creada",
    data: { ordenId: evento.data.ordenId, trackingNumber: guia.tracking },
  });
};

eventBus.on("orden.pagada", cuandoOrdenPagadaIniciarEnvio);
```

---

## 8.3 Aggregates que Emiten Eventos

Cada aggregate es una unidad de consistencia transaccional. Sus eventos son la **fuente de verdad**:

```typescript
// ─── Aggregate: Cliente ───
class Cliente {
  private id: string;
  private email: string;
  private nombre: string;
  private direcciones: DireccionEnvio[];
  private eventos: DomainEvent[] = [];

  static registrar(id: string, email: string, nombre: string): Cliente {
    const c = new Cliente();
    c.id = id;
    c.email = email;
    c.nombre = nombre;
    c.direcciones = [];

    c.agregarEvento({
      type: "cliente.registrado",
      data: { clienteId: id, email, nombre },
    });

    return c;
  }

  agregarDireccion(direccion: DireccionEnvio): void {
    this.direcciones.push(direccion);
    this.agregarEvento({
      type: "cliente.direccion_agregada",
      data: { clienteId: this.id, direccion },
    });
  }

  cambiarEmail(nuevoEmail: string): void {
    const anterior = this.email;
    this.email = nuevoEmail;

    this.agregarEvento({
      type: "cliente.email_cambiado",
      data: { clienteId: this.id, emailAnterior: anterior, emailNuevo: nuevoEmail },
    });
  }

  private agregarEvento(evento: DomainEvent): void {
    this.eventos.push({
      ...evento,
      metadata: {
        eventId: crypto.randomUUID(),
        timestamp: new Date().toISOString(),
        aggregateType: "Cliente",
        aggregateId: this.id,
        version: this.eventos.length + 1,
      },
    });
  }
}
```

---

## 8.4 Integration Events vs Domain Events

La diferencia es crucial cuando trabajamos con múltiples Bounded Contexts:

```typescript
// ─── DOMAIN EVENT (interno al contexto de Órdenes) ───
// Contiene TODA la información del dominio
namespace Ordenes {
  export interface OrdenPagada {
    type: "orden.pagada";
    data: {
      ordenId: string;
      metodo: string;
      transaccionId: string;
      monto: number;
      comisionProcesador: number;  // Detalle interno de órdenes
      metadataPago: Record<string, unknown>; // Datos completos
    };
  }
}

// ─── INTEGRATION EVENT (cruza fronteras de contexto) ───
// Contiene SOLO lo que otros contextos necesitan
namespace Integracion {
  export interface OrdenPagadaParaEnvio {
    type: "integracion.orden_pagada";
    data: {
      ordenId: string;
      clienteId: string;
      direccionEnvio: Direccion;
      montoTotal: number;
      // NO incluye comisionProcesador, metadataPago (internos)
    };
  }
}

// ─── Anti-Corruption Layer: traduce Domain Events a Integration Events ───
class IntegracionEventTranslator {
  traducirOrdenPagada(domainEvent: Ordenes.OrdenPagada, orden: Orden): Integracion.OrdenPagadaParaEnvio {
    return {
      type: "integracion.orden_pagada",
      data: {
        ordenId: domainEvent.data.ordenId,
        clienteId: orden.clienteId,
        direccionEnvio: orden.direccionEnvio,
        montoTotal: domainEvent.data.monto,
        // Solo los campos que el contexto de Envíos necesita
      },
    };
  }
}
```

### Anti-Corruption Layer (ACL)

```
┌──────────────────┐         ┌──────────────────┐
│ Bounded Context  │         │ Bounded Context  │
│     Órdenes      │         │      Envíos       │
│                  │         │                  │
│ Domain Events ───┼──ACL───▶│ Integration      │
│ (internos)       │         │ Events           │
│                  │         │ (modelo de envíos)│
└──────────────────┘         └──────────────────┘

ACL = traductor que protege cada contexto del modelo ajeno
```

```typescript
// Implementación de ACL
class EnviosAntiCorruptionLayer {
  constructor(
    private eventBus: EventBus,
    private servicioEnvios: ServicioEnvios,
  ) {
    // Escuchar eventos del contexto de Órdenes
    this.eventBus.on("orden.pagada", this.onOrdenPagada.bind(this));
  }

  private async onOrdenPagada(evento: Ordenes.OrdenPagada): Promise<void> {
    // Traducir al lenguaje de Envíos
    const solicitudEnvio: SolicitudEnvio = {
      referencia: evento.data.ordenId,
      destinatario: await this.obtenerDatosCliente(evento),
      paquetes: await this.calcularPaquetes(evento),
      fechaLimite: this.calcularFechaLimite(),
    };

    // Usar el modelo propio de Envíos
    await this.servicioEnvios.programarEnvio(solicitudEnvio);
  }
}
```

---

## 8.5 Bounded Contexts y Relaciones via Eventos

Los Bounded Contexts colaboran intercambiando eventos, no llamadas sincrónicas:

```
┌───────────────┐  OrdenCreada  ┌───────────────┐
│   Órdenes     │──────────────▶│   Inventario  │
│               │               │               │
│  Orden.crear()│               │  Al recibir   │
│  → OrdenCreada│               │  OrdenCreada: │
│               │               │  reservarItems│
└───────┬───────┘               └───────────────┘
        │
        │ OrdenCreada,
        │ OrdenPagada
        ▼
┌───────────────┐  OrdenEnviada ┌───────────────┐
│   Facturación │◀──────────────│    Envíos     │
│               │               │               │
│  Al recibir   │               │  Al recibir   │
│  OrdenPagada: │               │  OrdenPagada: │
│  generarFact  │               │  programarEnv │
└───────────────┘               └───────────────┘
```

### Relaciones Upstream/Downstream

```typescript
// ─── Contexto Upstream (Órdenes): publica eventos ───
// No sabe quién consume. No depende de nadie.
class OrdenesContexto {
  async procesarComando(comando: CrearOrden): Promise<void> {
    const orden = Orden.crear(comando);
    await this.repositorio.guardar(orden);

    // Publicar sin saber quién escucha
    for (const evento of orden.drenarEventos()) {
      await this.eventBus.publicar(evento);
    }
  }
}

// ─── Contexto Downstream (Facturación): se suscribe ───
// Depende del contrato de eventos de Órdenes
class FacturacionContexto {
  constructor(eventBus: EventBus) {
    eventBus.on("orden.pagada", this.onOrdenPagada.bind(this));
    eventBus.on("orden.cancelada", this.onOrdenCancelada.bind(this));
  }

  private async onOrdenPagada(evento: OrdenPagada): Promise<void> {
    // Guardar datos locales (Event Collaboration)
    await this.repositorio.guardarDatosOrden({
      ordenId: evento.data.ordenId,
      monto: evento.data.monto,
      metodo: evento.data.metodo,
      fecha: evento.metadata.timestamp,
    });

    // Emitir nuevo evento del contexto de Facturación
    await this.eventBus.publicar({
      type: "factura.generada",
      data: { facturaId: crypto.randomUUID(), ordenId: evento.data.ordenId },
    });
  }
}
```

---

## 8.6 Mapeo de Event Storming a Código

```typescript
// ─── Resultado del Event Storming ───
// Eventos: OrdenCreada, PagoProcesado, InventarioReservado, EnvioProgramado
// Comandos: CrearOrden, ProcesarPago, ReservarInventario, ProgramarEnvio
// Aggregates: Orden, Pago, Inventario, Envio
// Policies: "Cuando PagoProcesado → ReservarInventario"

// ─── Código que refleja el modelo ───

// Aggregate Orden
class Orden {
  // ... (ver sección 8.3)
}

// Policy (reacción automática)
const policyInventario: Policy = {
  nombre: "Reservar inventario cuando se procesa el pago",
  escucha: "pago.procesado",
  ejecuta: async (evento: PagoProcesado) => {
    const comando: ReservarInventarioComando = {
      ordenId: evento.data.ordenId,
      items: evento.data.items,
    };
    await comandos.enviar("inventario.reservar", comando);
  },
};

// Application Service (orquesta el flujo dentro del contexto)
class OrdenesApplicationService {
  async crearOrden(dto: CrearOrdenDTO): Promise<string> {
    const orden = Orden.crear(dto);
    await this.repositorio.guardar(orden);
    await this.eventBus.publicar(...orden.drenarEventos());
    return orden.id;
  }
}
```

---

## Resumen del Capítulo

- **Domain Events** representan cambios significativos en el negocio y son parte del lenguaje ubicuo.
- Los **aggregates** emiten eventos como resultado de comandos. Los eventos son la fuente de verdad del estado.
- **Event Storming** es una técnica colaborativa para descubrir eventos, comandos, aggregates, políticas y bounded contexts.
- **Integration Events** son versiones simplificadas de Domain Events, diseñadas para cruzar fronteras de bounded context.
- La **Anti-Corruption Layer** traduce eventos entre contextos, protegiendo cada modelo de dominio.
- Los **Bounded Contexts** colaboran exclusivamente mediante eventos. Upstream publica, downstream se suscribe.
- EDA + DDD permite modelar sistemas complejos donde cada contexto es autónomo y evoluciona independientemente.

En el siguiente capítulo exploramos Event Sourcing: almacenar eventos en lugar de estado.

---

← [Capítulo anterior](07-nats.md) | [Inicio](README.md) | [Capítulo siguiente →](09-event-sourcing.md)
