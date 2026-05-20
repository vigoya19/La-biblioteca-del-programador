# Capítulo 13: Sistema de E-Commerce Event-Driven

Este capítulo integra todos los conceptos del libro en una implementación completa de un sistema de e-commerce orientado a eventos. Construiremos el flujo completo: desde que un cliente crea una orden hasta que el producto es enviado.

> [!IMPORTANT]
> **Prerrequisitos de Lectura:**
> Esta implementación práctica está desarrollada en **TypeScript / Node.js** y está diseñada para ser empaquetada en contenedores. Para comprender al máximo los bloques de código y la configuración del entorno, te recomendamos encarecidamente revisar previamente:
> - El [Capítulo 7: Asincronía y el Event Loop](../node-programming-book/07-asincronia.md) del libro de **Node.js**.
> - El [Capítulo 4: El Archivo Dockerfile](../docker-programmin-book/capitulos/capitulo-04-dockerfile.md) del libro de **Docker**.

> "Un ejemplo vale más que mil diagramas de arquitectura." — Pragmatic Programmer

---

## 13.1 Arquitectura General

```
┌──────────────────────────────────────────────────────────────────┐
│                     E-COMMERCE EVENT-DRIVEN                       │
│                                                                  │
│  ┌──────────────┐      ┌──────────────┐      ┌──────────────┐    │
│  │   Órdenes    │      │   Pagos      │      │  Inventario  │    │
│  │   Service    │      │   Service    │      │   Service    │    │
│  │              │      │              │      │              │    │
│  │ POST /orden  │      │ on OrdenCreada      │ on PagoAprobado   │
│  │ → OrdenCreada│      │ → PagoProcesado     │ → StockReservado  │
│  └──────┬───────┘      └──────┬───────┘      └──────┬───────┘    │
│         │                     │                     │            │
│         └─────────────────────┼─────────────────────┘            │
│                               │                                  │
│                    ┌──────────▼──────────┐                       │
│                    │    EVENT BUS         │                       │
│                    │   (Kafka / RabbitMQ  │                       │
│                    │    / EventBridge)    │                       │
│                    └──────────┬──────────┘                       │
│                               │                                  │
│  ┌──────────────┐      ┌──────▼───────┐      ┌──────────────┐    │
│  │    Envíos    │      │ Notificaciones│      │  Facturación │    │
│  │   Service    │      │   Service     │      │   Service    │    │
│  │              │      │              │      │              │    │
│  │ on PagoAprobado     │ on OrdenEnviada     │ on PagoAprobado   │
│  │ → OrdenEnviada      │ → EmailEnviado      │ → FacturaEmitida  │
│  └──────────────┘      └──────────────┘      └──────────────┘    │
└──────────────────────────────────────────────────────────────────┘
```

### Eventos del Sistema

| Evento | Productor | Consumidores |
|--------|-----------|-------------|
| `orden.creada` | Servicio Órdenes | Pagos, Analytics |
| `pago.procesado` | Servicio Pagos | Inventario, Notificaciones, Facturación |
| `pago.rechazado` | Servicio Pagos | Órdenes, Notificaciones |
| `inventario.reservado` | Servicio Inventario | Notificaciones |
| `inventario.no_disponible` | Servicio Inventario | Órdenes, Pagos (compensar), Notificaciones |
| `orden.enviada` | Servicio Envíos | Notificaciones, Analytics |
| `factura.emitida` | Servicio Facturación | Notificaciones |

---

## 13.2 Setup del Proyecto

```bash
# Estructura del proyecto
ecommerce-event-driven/
├── packages/
│   ├── shared/              # Tipos de eventos, utils compartidos
│   │   ├── eventos.ts
│   │   ├── eventBus.ts
│   │   └── idempotencia.ts
│   ├── ordenes/             # Servicio de Órdenes
│   │   ├── index.ts
│   │   ├── handlers.ts
│   │   └── modelo.ts
│   ├── pagos/               # Servicio de Pagos
│   │   ├── index.ts
│   │   └── handlers.ts
│   ├── inventario/          # Servicio de Inventario
│   │   ├── index.ts
│   │   └── handlers.ts
│   ├── envios/              # Servicio de Envíos
│   │   ├── index.ts
│   │   └── handlers.ts
│   ├── notificaciones/      # Servicio de Notificaciones
│   │   └── index.ts
│   └── facturacion/         # Servicio de Facturación
│       └── index.ts
```

### Tipos Compartidos (packages/shared/eventos.ts)

```typescript
// ─── Event Metadata ───
export interface EventMetadata {
  eventId: string;
  timestamp: string;
  correlationId: string;
  causationId?: string;
  aggregateType: string;
  aggregateId: string;
  version: number;
}

// ─── Domain Events ───
export interface OrdenCreada {
  type: "orden.creada";
  data: {
    ordenId: string;
    clienteId: string;
    emailCliente: string;
    items: { productoId: string; nombre: string; cantidad: number; precioUnitario: number }[];
    total: number;
    direccionEnvio: {
      calle: string;
      ciudad: string;
      codigoPostal: string;
      pais: string;
    };
  };
  metadata: EventMetadata;
}

export interface PagoProcesado {
  type: "pago.procesado";
  data: {
    pagoId: string;
    ordenId: string;
    metodo: "tarjeta" | "transferencia";
    monto: number;
    moneda: string;
    transaccionId: string;
  };
  metadata: EventMetadata;
}

export interface PagoRechazado {
  type: "pago.rechazado";
  data: {
    pagoId: string;
    ordenId: string;
    motivo: string;
  };
  metadata: EventMetadata;
}

export interface InventarioReservado {
  type: "inventario.reservado";
  data: {
    reservaId: string;
    ordenId: string;
    items: { productoId: string; cantidad: number }[];
  };
  metadata: EventMetadata;
}

export interface InventarioNoDisponible {
  type: "inventario.no_disponible";
  data: {
    ordenId: string;
    items: { productoId: string; cantidadSolicitada: number; stockDisponible: number }[];
  };
  metadata: EventMetadata;
}

export interface OrdenEnviada {
  type: "orden.enviada";
  data: {
    ordenId: string;
    transportista: string;
    trackingNumber: string;
    fechaEstimadaEntrega: string;
  };
  metadata: EventMetadata;
}

export interface FacturaEmitida {
  type: "factura.emitida";
  data: {
    facturaId: string;
    ordenId: string;
    clienteId: string;
    total: number;
    pdfUrl: string;
  };
  metadata: EventMetadata;
}
```

### Event Bus Abstraction (packages/shared/eventBus.ts)

```typescript
type EventHandler<T extends DomainEvent = DomainEvent> = (evento: T) => Promise<void>;

interface EventBus {
  publicar<T extends DomainEvent>(evento: T): Promise<void>;
  suscribir<T extends DomainEvent>(eventType: string, handler: EventHandler<T>): Promise<void>;
}

// Implementación con Kafka (extensible a RabbitMQ, EventBridge, etc.)
class KafkaEventBus implements EventBus {
  constructor(
    private producer: KafkaProducer,
    private consumer: KafkaConsumer,
  ) {}

  async publicar<T extends DomainEvent>(evento: T): Promise<void> {
    await this.producer.send({
      topic: evento.metadata.aggregateType.toLowerCase(),
      messages: [{
        key: evento.metadata.aggregateId,
        value: JSON.stringify(evento),
        headers: { "event-type": evento.type },
      }],
    });
  }

  async suscribir<T extends DomainEvent>(
    eventType: string,
    handler: EventHandler<T>,
  ): Promise<void> {
    await this.consumer.subscribe({ topics: this.inferirTopics(eventType) });

    this.consumer.run({
      eachMessage: async ({ message }) => {
        const evento = JSON.parse(message.value!.toString()) as T;
        if (evento.type === eventType) {
          await handler(evento);
        }
      },
    });
  }

  private inferirTopics(eventType: string): string[] {
    const [aggregate] = eventType.split(".");
    return [aggregate.toLowerCase()];
  }
}
```

---

## 13.3 Servicio de Órdenes

```typescript
// packages/ordenes/modelo.ts
class Orden {
  readonly eventos: DomainEvent[] = [];

  private constructor(
    public readonly ordenId: string,
    public readonly clienteId: string,
    public readonly emailCliente: string,
    public readonly items: ItemOrden[],
    public readonly total: number,
    public readonly direccionEnvio: DireccionEnvio,
    public estado: "pendiente" | "pagada" | "enviada" | "cancelada" = "pendiente",
  ) {}

  static crear(dto: CrearOrdenDTO): Orden {
    const total = dto.items.reduce((sum, i) => sum + i.cantidad * i.precioUnitario, 0);

    const orden = new Orden(
      crypto.randomUUID(),
      dto.clienteId,
      dto.emailCliente,
      dto.items.map(i => new ItemOrden(i.productoId, i.nombre, i.cantidad, i.precioUnitario)),
      total,
      dto.direccionEnvio,
      "pendiente",
    );

    orden.agregarEvento<OrdenCreada>({
      type: "orden.creada",
      data: {
        ordenId: orden.ordenId,
        clienteId: orden.clienteId,
        emailCliente: orden.emailCliente,
        items: orden.items.map(i => i.toDTO()),
        total: orden.total,
        direccionEnvio: orden.direccionEnvio,
      },
    });

    return orden;
  }

  cancelar(motivo: string): void {
    if (this.estado === "enviada") {
      throw new Error("No se puede cancelar una orden ya enviada");
    }
    this.estado = "cancelada";

    this.agregarEvento({
      type: "orden.cancelada",
      data: { ordenId: this.ordenId, motivo },
    });
  }

  private agregarEvento<T extends DomainEvent>(
    evento: Omit<T, "metadata">,
  ): void {
    const eventoConMetadata = {
      ...evento,
      metadata: {
        eventId: crypto.randomUUID(),
        timestamp: new Date().toISOString(),
        correlationId: "trace-" + this.ordenId,
        aggregateType: "Orden",
        aggregateId: this.ordenId,
        version: this.eventos.length + 1,
      },
    } as T;

    this.eventos.push(eventoConMetadata);
  }
}

// packages/ordenes/handlers.ts
class OrdenesHandlers {
  constructor(
    private repositorio: OrdenRepository,
    private eventBus: EventBus,
  ) {}

  // ─── API Handler ───
  async crearOrden(req: Request, res: Response): Promise<void> {
    const dto: CrearOrdenDTO = req.body;

    try {
      const orden = Orden.crear(dto);

      // Persistir orden
      await this.repositorio.guardar(orden);

      // Publicar eventos
      for (const evento of orden.eventos) {
        await this.eventBus.publicar(evento);
      }

      res.status(202).json({
        ordenId: orden.ordenId,
        estado: orden.estado,
        _links: { consultar: `/ordenes/${orden.ordenId}` },
      });
    } catch (error) {
      res.status(400).json({ error: (error as Error).message });
    }
  }

  // ─── Event Handlers (reacciones) ───
  async onPagoRechazado(evento: PagoRechazado): Promise<void> {
    const orden = await this.repositorio.obtener(evento.data.ordenId);
    if (!orden) return;

    orden.cancelar(`Pago rechazado: ${evento.data.motivo}`);
    await this.repositorio.actualizar(orden);

    for (const evt of orden.eventos) {
      await this.eventBus.publicar(evt);
    }
  }

  async onInventarioNoDisponible(evento: InventarioNoDisponible): Promise<void> {
    const orden = await this.repositorio.obtener(evento.data.ordenId);
    if (!orden || orden.estado === "cancelada") return;

    const detalle = evento.data.items
      .map(i => `${i.productoId}: solicitado ${i.cantidadSolicitada}, stock ${i.stockDisponible}`)
      .join("; ");

    orden.cancelar(`Inventario insuficiente: ${detalle}`);
    await this.repositorio.actualizar(orden);

    for (const evt of orden.eventos) {
      await this.eventBus.publicar(evt);
    }
  }
}
```

---

## 13.4 Servicio de Pagos

```typescript
// packages/pagos/handlers.ts
class PagosHandlers {
  constructor(
    private procesador: ProcesadorPagos,
    private eventBus: EventBus,
  ) {}

  @onEvent("orden.creada")
  async onOrdenCreada(evento: OrdenCreada): Promise<void> {
    const pagoId = crypto.randomUUID();

    try {
      logger.info("Procesando pago", { ordenId: evento.data.ordenId, total: evento.data.total });

      const resultado = await this.procesador.cobrar({
        clienteId: evento.data.clienteId,
        monto: evento.data.total,
        metodo: "tarjeta",
        idempotencyKey: pagoId, // El banco sabe que este pago es único
      });

      if (resultado.exitoso) {
        await this.eventBus.publicar({
          type: "pago.procesado",
          data: {
            pagoId,
            ordenId: evento.data.ordenId,
            metodo: "tarjeta",
            monto: evento.data.total,
            moneda: "EUR",
            transaccionId: resultado.transaccionId,
          },
          metadata: {
            eventId: crypto.randomUUID(),
            timestamp: new Date().toISOString(),
            correlationId: evento.metadata.correlationId,
            causationId: evento.metadata.eventId,
            aggregateType: "Pago",
            aggregateId: pagoId,
            version: 1,
          },
        });
      } else {
        await this.eventBus.publicar({
          type: "pago.rechazado",
          data: {
            pagoId,
            ordenId: evento.data.ordenId,
            motivo: resultado.motivo ?? "Rechazado por el banco",
          },
          metadata: {
            eventId: crypto.randomUUID(),
            timestamp: new Date().toISOString(),
            correlationId: evento.metadata.correlationId,
            causationId: evento.metadata.eventId,
            aggregateType: "Pago",
            aggregateId: pagoId,
            version: 1,
          },
        });
      }
    } catch (error) {
      // Error técnico (timeout, red): reintentar
      logger.error("Error procesando pago, reintentando", { error, ordenId: evento.data.ordenId });
    }
  }
}
```

---

## 13.5 Servicio de Inventario

```typescript
// packages/inventario/handlers.ts
class InventarioHandlers {
  private reservasProcesadas = new Set<string>(); // Idempotencia

  @onEvent("pago.procesado")
  async onPagoProcesado(evento: PagoProcesado): Promise<void> {
    // Idempotencia
    if (this.reservasProcesadas.has(evento.metadata.eventId)) {
      logger.warn("Evento de pago duplicado, ignorando", { eventId: evento.metadata.eventId });
      return;
    }

    // Obtener los items de la orden desde su proyección local
    const itemsOrden = await this.obtenerItemsOrden(evento.data.ordenId);

    try {
      const reservaId = crypto.randomUUID();
      const noDisponibles: { productoId: string; cantidadSolicitada: number; stockDisponible: number }[] = [];

      // Verificar stock y reservar
      for (const item of itemsOrden) {
        const reservado = await this.reservarProducto(item.productoId, item.cantidad);
        if (!reservado) {
          const stock = await this.obtenerStock(item.productoId);
          noDisponibles.push({ productoId: item.productoId, cantidadSolicitada: item.cantidad, stockDisponible: stock });
        }
      }

      if (noDisponibles.length > 0) {
        // Compensar: liberar lo ya reservado
        await this.liberarProductos(reservaId);
        await this.eventBus.publicar({
          type: "inventario.no_disponible",
          data: { ordenId: evento.data.ordenId, items: noDisponibles },
          metadata: this.crearMetadata(evento, "Inventario", reservaId),
        });
        return;
      }

      await this.eventBus.publicar({
        type: "inventario.reservado",
        data: { reservaId, ordenId: evento.data.ordenId, items: itemsOrden },
        metadata: this.crearMetadata(evento, "Inventario", reservaId),
      });

      this.reservasProcesadas.add(evento.metadata.eventId);
    } catch (error) {
      logger.error("Error en reserva de inventario", { error, ordenId: evento.data.ordenId });
    }
  }

  async reservarProducto(productoId: string, cantidad: number): Promise<boolean> {
    const { rows } = await this.db.query(
      `UPDATE productos
       SET stock = stock - $2
       WHERE producto_id = $1 AND stock >= $2
       RETURNING producto_id`,
      [productoId, cantidad],
    );
    return rows.length > 0;
  }

  async liberarProductos(reservaId: string): Promise<void> {
    // Lógica de compensación: devolver stock
  }
}
```

---

## 13.6 Servicio de Envíos

```typescript
// packages/envios/handlers.ts
class EnviosHandlers {
  @onEvent("pago.procesado")
  async onPagoProcesado(evento: PagoProcesado): Promise<void> {
    // Obtener dirección de envío desde proyección de órdenes
    const datosEnvio = await this.obtenerDatosEnvio(evento.data.ordenId);

    // Integración con transportista
    const guia = await this.transportista.crearEnvio({
      ordenId: evento.data.ordenId,
      direccion: datosEnvio.direccion,
      paquetes: datosEnvio.paquetes,
    });

    await this.eventBus.publicar({
      type: "orden.enviada",
      data: {
        ordenId: evento.data.ordenId,
        transportista: guia.transportista,
        trackingNumber: guia.trackingNumber,
        fechaEstimadaEntrega: guia.fechaEstimada,
      },
      metadata: this.crearMetadata(evento, "Envio", guia.guiaId),
    });
  }
}
```

---

## 13.7 Servicio de Notificaciones

```typescript
// packages/notificaciones/index.ts
class NotificacionesService {
  @onEvent("orden.creada")
  async onOrdenCreada(evento: OrdenCreada): Promise<void> {
    await this.email.enviar({
      destinatario: evento.data.emailCliente,
      asunto: `Tu orden #${evento.data.ordenId.slice(0, 8)} ha sido creada`,
      cuerpo: `Hola, tu orden por ${evento.data.total} EUR está siendo procesada.`,
    });
  }

  @onEvent("pago.procesado")
  async onPagoProcesado(evento: PagoProcesado): Promise<void> {
    // Notificación de pago exitoso
  }

  @onEvent("pago.rechazado")
  async onPagoRechazado(evento: PagoRechazado): Promise<void> {
    const orden = await this.obtenerProyeccionOrden(evento.data.ordenId);
    await this.email.enviar({
      destinatario: orden.emailCliente,
      asunto: "Problema con tu pago",
      cuerpo: `Tu pago fue rechazado: ${evento.data.motivo}. Por favor intenta de nuevo.`,
    });
  }

  @onEvent("inventario.no_disponible")
  async onInventarioNoDisponible(evento: InventarioNoDisponible): Promise<void> {
    const orden = await this.obtenerProyeccionOrden(evento.data.ordenId);
    await this.email.enviar({
      destinatario: orden.emailCliente,
      asunto: "Productos no disponibles en tu orden",
      cuerpo: `Algunos productos de tu orden no tienen stock suficiente.`,
    });
  }

  @onEvent("orden.enviada")
  async onOrdenEnviada(evento: OrdenEnviada): Promise<void> {
    // Email con tracking number
  }

  @onEvent("factura.emitida")
  async onFacturaEmitida(evento: FacturaEmitida): Promise<void> {
    // Email con factura PDF adjunta
  }
}
```

---

## 13.8 Bootstrapping del Sistema

```typescript
// packages/main/index.ts
async function iniciarSistema(): Promise<void> {
  const eventBus = new KafkaEventBus(producer, consumer);

  // ─── Instanciar servicios ───
  const ordenesRepo = new OrdenRepository(db);
  const ordenesHandlers = new OrdenesHandlers(ordenesRepo, eventBus);
  const pagosHandlers = new PagosHandlers(procesadorPagos, eventBus);
  const inventarioHandlers = new InventarioHandlers(db, eventBus);
  const enviosHandlers = new EnviosHandlers(transportista, db, eventBus);
  const notificacionesService = new NotificacionesService(email, db);
  const facturacionService = new FacturacionService(db, eventBus);

  // ─── Suscribir handlers a eventos ───

  // Pagos escucha órdenes creadas
  await eventBus.suscribir("orden.creada", pagosHandlers.onOrdenCreada.bind(pagosHandlers));

  // Inventario escucha pagos procesados
  await eventBus.suscribir("pago.procesado", inventarioHandlers.onPagoProcesado.bind(inventarioHandlers));

  // Envíos escucha pagos procesados
  await eventBus.suscribir("pago.procesado", enviosHandlers.onPagoProcesado.bind(enviosHandlers));

  // Órdenes escucha fallos (compensación)
  await eventBus.suscribir("pago.rechazado", ordenesHandlers.onPagoRechazado.bind(ordenesHandlers));
  await eventBus.suscribir("inventario.no_disponible", ordenesHandlers.onInventarioNoDisponible.bind(ordenesHandlers));

  // Notificaciones escucha todo
  for (const tipo of ["orden.creada", "pago.procesado", "pago.rechazado", "inventario.no_disponible", "orden.enviada", "factura.emitida"]) {
    await eventBus.suscribir(tipo, (e: DomainEvent) => notificacionesService.handle(e));
  }

  // Facuración escucha pagos
  await eventBus.suscribir("pago.procesado", facturacionService._onPagoProcesado.bind(facturacionService));

  // ─── API HTTP ───
  const app = express();
  app.post("/api/ordenes", ordenesHandlers.crearOrden.bind(ordenesHandlers));
  app.get("/api/ordenes/:id", ordenesHandlers.consultarOrden.bind(ordenesHandlers));

  app.listen(3000, () => {
    logger.info("Sistema E-Commerce Event-Driven iniciado en :3000");
  });
}

iniciarSistema().catch(logger.error);
```

---

## 13.9 Flujo Completo: Happy Path

```
1. POST /api/ordenes { clienteId, items, direccionEnvio }
   └─▶ Servicio Órdenes: Orden.crear()
       ├─ Guarda en BD de órdenes
       └─ Publica: OrdenCreada

2. Servicio Pagos recibe OrdenCreada
   ├─ Procesa cobro con banco
   └─ Publica: PagoProcesado (o PagoRechazado)

3a. Servicio Inventario recibe PagoProcesado   3b. Servicio Envíos recibe PagoProcesado
    ├─ Reserva stock                                 ├─ Crea guía con transportista
    └─ Publica: InventarioReservado                  └─ Publica: OrdenEnviada
        (o InventarioNoDisponible)

4. Servicio Facturación recibe PagoProcesado
   ├─ Genera factura PDF
   └─ Publica: FacturaEmitida

5. Servicio Notificaciones recibe todos los eventos
   ├─ OrdenCreada → Email: "Orden recibida"
   ├─ PagoProcesado → Email: "Pago confirmado"
   ├─ OrdenEnviada → Email: "Tu pedido va en camino" + tracking
   └─ FacturaEmitida → Email: "Tu factura" + PDF adjunto
```

---

## 13.10 Flujo con Fallo: Compensación

```
1. POST /api/ordenes → OrdenCreada
2. Pagos: PagoProcesado ✅
3. Inventario: producto AGOTADO ❌
   └─ Publica: InventarioNoDisponible

4. Servicio Órdenes recibe InventarioNoDisponible
   ├─ Cancela la orden
   └─ Publica: OrdenCancelada

5. Servicio Pagos recibe OrdenCancelada
   ├─ Reembolsa el pago al cliente
   └─ Publica: PagoReembolsado

6. Notificaciones:
   ├─ InventarioNoDisponible → Email: "Producto sin stock"
   └─ OrdenCancelada → Email: "Tu orden fue cancelada"
```

---

## Resumen del Capítulo

- El sistema de e-commerce implementa todos los patrones del libro: **Domain Events, coreografía, proyecciones, idempotencia, sagas de compensación.**
- Cada servicio es autónomo: tiene su propia BD, su propia lógica, y se comunica solo mediante eventos.
- **Happy path**: OrdenCreada → PagoProcesado → InventarioReservado + OrdenEnviada + FacturaEmitida.
- **Fallo**: InventarioNoDisponible → OrdenCancelada → PagoReembolsado.
- La **idempotencia** se implementa con eventId + Set de procesados.
- **CausationId** permite trazar qué evento causó qué respuesta (trazas distribuidas).

En el siguiente capítulo abordamos la evolución de esquemas de eventos.
