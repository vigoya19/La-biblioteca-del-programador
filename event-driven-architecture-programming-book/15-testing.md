# Capítulo 15: Testing en Sistemas Event-Driven

Testear sistemas orientados a eventos requiere un enfoque diferente al testing tradicional. Los componentes son asíncronos, distribuidos y con consistencia eventual. La pirámide de testing se adapta para incluir la capa de mensajería.

> "Si no puedes probarlo, no puedes desplegarlo con confianza." — Principio de Testabilidad

---

## 15.1 Test Pyramid para Sistemas de Eventos

```
┌──────────────────────────────────────────────┐
│              PIRÁMIDE DE TESTING EDA          │
│                                              │
│                    /\                         │
│                   /E2E\        Pocos, lentos │
│                  /──────\                     │
│                 /Contract\     Consumer-Driven│
│                /  Tests  \     Contract Tests │
│               /──────────\                    │
│              /Integration \   Con Testcontainers│
│             /   Tests      \                  │
│            /────────────────\                 │
│           /                  \                │
│          /   Unit Tests       \  Muchos, rápidos│
│         /  (Handlers, Models) \               │
│        /────────────────────────\              │
└──────────────────────────────────────────────┘
```

---

## 15.2 Unit Testing de Handlers de Eventos

El handler es la unidad mínima en EDA. Se prueba de forma aislada, con mocks para dependencias externas.

```typescript
// ─── Handler a testear ───
class PagosHandler {
  constructor(
    private procesadorPagos: ProcesadorPagos,
    private eventBus: EventBus,
    private logger: Logger,
  ) {}

  async onOrdenCreada(evento: OrdenCreada): Promise<void> {
    this.logger.info("Procesando pago para orden", { ordenId: evento.data.ordenId });

    if (evento.data.total <= 0) {
      this.logger.error("Total inválido", { total: evento.data.total });
      return;
    }

    const resultado = await this.procesadorPagos.cobrar({
      clienteId: evento.data.clienteId,
      monto: evento.data.total,
      metodo: "tarjeta",
    });

    if (resultado.exitoso) {
      await this.eventBus.publicar({
        type: "pago.procesado",
        data: {
          pagoId: crypto.randomUUID(),
          ordenId: evento.data.ordenId,
          metodo: "tarjeta",
          monto: evento.data.total,
          transaccionId: resultado.transaccionId,
        },
      });
    } else {
      await this.eventBus.publicar({
        type: "pago.rechazado",
        data: {
          pagoId: crypto.randomUUID(),
          ordenId: evento.data.ordenId,
          motivo: resultado.motivo,
        },
      });
    }
  }
}

// ─── Unit Tests ───
import { mock, when, instance, verify, anything } from "ts-mockito";

describe("PagosHandler.onOrdenCreada", () => {
  let handler: PagosHandler;
  let procesadorMock: ProcesadorPagos;
  let eventBusMock: EventBus;
  let loggerMock: Logger;

  const eventoBase: OrdenCreada = {
    type: "orden.creada",
    data: {
      ordenId: "ord-123",
      clienteId: "cli-456",
      items: [{ productoId: "P1", cantidad: 2, precio: 49.99 }],
      total: 99.98,
      direccionEnvio: { calle: "Mayor 1", ciudad: "Madrid", codigoPostal: "28013" },
      emailCliente: "test@test.com",
    },
    metadata: {
      eventId: "evt-001",
      timestamp: new Date().toISOString(),
      correlationId: "trace-001",
      aggregateType: "Orden",
      aggregateId: "ord-123",
      version: 1,
    },
  };

  beforeEach(() => {
    procesadorMock = mock<ProcesadorPagos>();
    eventBusMock = mock<EventBus>();
    loggerMock = mock<Logger>();
    handler = new PagosHandler(instance(procesadorMock), instance(eventBusMock), instance(loggerMock));
  });

  test("debe publicar PagoProcesado cuando el cobro es exitoso", async () => {
    when(procesadorMock.cobrar(anything())).thenResolve({
      exitoso: true,
      transaccionId: "txn-001",
    });

    await handler.onOrdenCreada(eventoBase);

    verify(eventBusMock.publicar(anything())).once();
  });

  test("debe publicar PagoRechazado cuando el cobro falla", async () => {
    when(procesadorMock.cobrar(anything())).thenResolve({
      exitoso: false,
      motivo: "Fondos insuficientes",
    });

    await handler.onOrdenCreada(eventoBase);

    verify(eventBusMock.publicar(anything())).once();
  });

  test("NO debe publicar evento si el total es inválido", async () => {
    const eventoInvalido = {
      ...eventoBase,
      data: { ...eventoBase.data, total: 0 },
    };

    await handler.onOrdenCreada(eventoInvalido);

    verify(eventBusMock.publicar(anything())).never();
    verify(procesadorMock.cobrar(anything())).never();
  });

  test("debe registrar el inicio del procesamiento", async () => {
    when(procesadorMock.cobrar(anything())).thenResolve({ exitoso: true, transaccionId: "txn-001" });

    await handler.onOrdenCreada(eventoBase);

    verify(loggerMock.info("Procesando pago para orden", anything())).once();
  });
});
```

### Test de Aggregate (Event Sourcing)

```typescript
describe("Orden aggregate", () => {
  test("crear emite OrdenCreada con los datos correctos", () => {
    const dto: CrearOrdenDTO = {
      clienteId: "cli-456",
      emailCliente: "test@test.com",
      items: [{ productoId: "P1", nombre: "Libro", cantidad: 2, precioUnitario: 49.99 }],
      direccionEnvio: { calle: "Mayor 1", ciudad: "Madrid", codigoPostal: "28013", pais: "ES" },
    };

    const orden = Orden.crear(dto);

    expect(orden.eventos).toHaveLength(1);
    expect(orden.eventos[0].type).toBe("orden.creada");
    expect(orden.eventos[0].data.total).toBe(99.98);
    expect(orden.eventos[0].data.clienteId).toBe("cli-456");
    expect(orden.estado).toBe("pendiente");
  });

  test("reconstruirDesde reproduce el estado correctamente", () => {
    const historial: DomainEvent[] = [
      crearEvento("orden.creada", { ordenId: "ord-1", clienteId: "cli-1", total: 100 }),
      crearEvento("orden.pagada", { ordenId: "ord-1", metodo: "tarjeta", monto: 100 }),
    ];

    const orden = Orden.reconstruirDesde(historial);

    expect(orden.estado).toBe("pagada");
  });

  test("no permite pagar una orden ya cancelada", () => {
    const historial: DomainEvent[] = [
      crearEvento("orden.creada", { ordenId: "ord-1", clienteId: "cli-1", total: 100 }),
      crearEvento("orden.cancelada", { ordenId: "ord-1", motivo: "fraude" }),
    ];
    const orden = Orden.reconstruirDesde(historial);

    expect(() => orden.pagar("tarjeta", "txn-1", 100))
      .toThrow("No se puede pagar");
  });
});
```

### Test de Proyección

```typescript
describe("ProyeccionOrdenesConsulta", () => {
  let proyeccion: ProyeccionOrdenesConsulta;
  let db: Database;

  beforeAll(async () => {
    db = await createTestDatabase();
    proyeccion = new ProyeccionOrdenesConsulta(db);
  });

  beforeEach(async () => {
    await db.query("TRUNCATE ordenes_consulta");
  });

  test("onOrdenCreada inserta registro en tabla de consulta", async () => {
    await proyeccion.onOrdenCreada(eventoOrdenCreada);

    const { rows } = await db.query(
      "SELECT * FROM ordenes_consulta WHERE orden_id = $1",
      ["ord-123"],
    );

    expect(rows).toHaveLength(1);
    expect(rows[0].estado).toBe("creada");
    expect(Number(rows[0].total)).toBe(99.98);
  });

  test("onOrdenPagada actualiza estado de la orden", async () => {
    await proyeccion.onOrdenCreada(eventoOrdenCreada);
    await proyeccion.onOrdenPagada(eventoOrdenPagada);

    const { rows } = await db.query(
      "SELECT estado, metodo_pago FROM ordenes_consulta WHERE orden_id = $1",
      ["ord-123"],
    );

    expect(rows[0].estado).toBe("pagada");
    expect(rows[0].metodo_pago).toBe("tarjeta");
  });
});
```

---

## 15.3 Integration Testing con Testcontainers

Testcontainers permite levantar infraestructura real (Kafka, PostgreSQL, RabbitMQ) en contenedores Docker para tests de integración.

```typescript
import { KafkaContainer, StartedKafkaContainer } from "@testcontainers/kafka";
import { PostgreSqlContainer } from "@testcontainers/postgresql";
import { Kafka, Producer, Consumer } from "kafkajs";

describe("Integración: Event Bus + Handler", () => {
  let kafkaContainer: StartedKafkaContainer;
  let postgresContainer: StartedPostgreSqlContainer;
  let eventBus: KafkaEventBus;
  let producer: Producer;
  let consumer: Consumer;

  beforeAll(async () => {
    // Levantar Kafka
    kafkaContainer = await new KafkaContainer().start();

    // Levantar PostgreSQL
    postgresContainer = await new PostgreSqlContainer()
      .withDatabase("testdb")
      .start();

    // Configurar clientes
    const kafka = new Kafka({
      brokers: [kafkaContainer.getBootstrapServer()],
    });
    producer = kafka.producer();
    consumer = kafka.consumer({ groupId: "test-group" });

    await producer.connect();
    await consumer.connect();
  }, 60000);

  afterAll(async () => {
    await producer.disconnect();
    await consumer.disconnect();
    await kafkaContainer.stop();
    await postgresContainer.stop();
  });

  test("flujo completo: OrdenCreada → PagoProcesado", async () => {
    const pagosHandler = new PagosHandler(
      new ProcesadorPagosMock(),
      eventBus,
      logger,
    );

    // Suscribir handler
    await eventBus.suscribir("orden.creada", pagosHandler.onOrdenCreada.bind(pagosHandler));

    // Array para capturar eventos publicados
    const eventosPublicados: DomainEvent[] = [];
    await consumer.subscribe({ topic: "pagos", fromBeginning: true });
    consumer.run({
      eachMessage: async ({ message }) => {
        eventosPublicados.push(JSON.parse(message.value!.toString()));
      },
    });

    // Act: publicar evento de orden creada
    const evento: OrdenCreada = { /* ... */ };
    await eventBus.publicar(evento);

    // Assert: esperar que PagoProcesado sea publicado
    await new Promise(resolve => setTimeout(resolve, 2000));

    expect(eventosPublicados).toHaveLength(1);
    expect(eventosPublicados[0].type).toBe("pago.procesado");
    expect(eventosPublicados[0].data.ordenId).toBe("ord-123");
    expect(eventosPublicados[0].data.monto).toBe(99.98);
  });

  test("idempotencia: evento duplicado no duplica procesamiento", async () => {
    // Publicar mismo evento dos veces
    const evento: OrdenCreada = { /* ... */ };
    await eventBus.publicar(evento);
    await eventBus.publicar(evento);

    await new Promise(resolve => setTimeout(resolve, 2000));

    // Solo se procesa una vez
    expect(procesadosEnBD).toBe(1);
  });
});
```

---

## 15.4 Consumer-Driven Contract Testing

Los contratos entre servicios se validan con Pact. El consumidor define qué espera; el productor verifica que cumple.

```typescript
// ─── Consumidor: Servicio de Pagos ───
// Define el contrato: "cuando recibo OrdenCreada, espero este formato"

import { Pact, Matchers } from "@pact-foundation/pact";

const provider = new Pact({
  consumer: "servicio-pagos",
  provider: "servicio-ordenes",
  logLevel: "warn",
});

describe("Pact: Pagos consume OrdenCreada", () => {
  beforeAll(() => provider.setup());

  test("espera un evento OrdenCreada válido", async () => {
    await provider.addInteraction({
      state: "existe una orden creada",
      uponReceiving: "un evento OrdenCreada",
      withRequest: {
        // El evento que el consumidor espera recibir
        body: {
          type: "orden.creada",
          data: {
            ordenId: Matchers.string("ord-123"),
            clienteId: Matchers.string("cli-456"),
            total: Matchers.decimal(99.98),
            items: Matchers.eachLike({
              productoId: Matchers.string("P1"),
              cantidad: Matchers.integer(2),
              precio: Matchers.decimal(49.99),
            }),
            direccionEnvio: Matchers.like({
              calle: "Mayor 1",
              ciudad: "Madrid",
              codigoPostal: "28013",
            }),
          },
          metadata: {
            eventId: Matchers.string("evt-001"),
            timestamp: Matchers.iso8601(),
          },
        },
      },
      willRespondWith: { status: 200 },
    });

    // El consumidor valida que puede procesar el evento
    const handler = new PagosHandler(procesador, eventBus);
    await handler.onOrdenCreada(eventoDePrueba);
    // No debe lanzar errores
  });

  afterAll(() => provider.finalize());
});
```

### Contract Testing con Schema Registry

```typescript
// Validar que el schema del productor es compatible con lo que espera el consumidor

import { SchemaRegistry } from "@kafkajs/confluent-schema-registry";

describe("Schema Compatibility", () => {
  const registry = new SchemaRegistry({ host: "http://schema-registry:8081" });

  test("OrdenCreada v2 es backward compatible con v1", async () => {
    const result = await registry.testCompatibility(
      "ordenes-value",
      schemaOrdenCreadaV2,
    );

    expect(result.isCompatible).toBe(true);
  });

  test("Schema del productor cumple el contrato del consumidor", async () => {
    const schemaProductor = await registry.getLatestSchema("ordenes-value");

    // Validar que el consumidor puede deserializar el schema del productor
    const eventBytes = await registry.encode(100, eventoEjemplo);
    const decoded = await registry.decode(eventBytes);

    expect(decoded.ordenId).toBeDefined();
    expect(decoded.clienteId).toBeDefined();
    expect(typeof decoded.total).toBe("number");
  });
});
```

---

## 15.5 Testing de Sagas

```typescript
describe("ProcesarOrdenSaga - Flujo completo", () => {
  let saga: ProcesarOrdenSagaOrquestador;
  let eventBus: InMemoryEventBus; // Bus en memoria para tests

  beforeEach(() => {
    eventBus = new InMemoryEventBus();
    saga = new ProcesarOrdenSagaOrquestador(eventBus, db);
  });

  test("saga exitosa: todos los pasos se completan", async () => {
    const sagaId = await saga.iniciar({
      clienteId: "cli-1",
      items: [{ productoId: "P1", cantidad: 2, precioUnitario: 50 }],
    });

    // Simular respuestas de servicios
    await eventBus.publicar({ type: "orden.creada", metadata: { causationId: sagaId } });
    await eventBus.publicar({ type: "inventario.reservado", metadata: { causationId: sagaId } });
    await eventBus.publicar({ type: "pago.procesado", metadata: { causationId: sagaId } });

    const estado = await saga.obtenerEstado(sagaId);
    expect(estado.pasoActual).toBe("completada");
    expect(estado.historial).toHaveLength(3);
    expect(estado.historial.every(h => h.exitoso)).toBe(true);
  });

  test("saga compensa cuando el pago es rechazado", async () => {
    const sagaId = await saga.iniciar({
      clienteId: "cli-1",
      items: [{ productoId: "P1", cantidad: 2, precioUnitario: 50 }],
    });

    // Paso 1 y 2 exitosos
    await eventBus.publicar({ type: "orden.creada", metadata: { causationId: sagaId } });
    await eventBus.publicar({ type: "inventario.reservado", metadata: { causationId: sagaId } });

    // Paso 3 falla
    const comandosCompensacion = capturarComandos();
    await eventBus.publicar({
      type: "pago.rechazado",
      metadata: { causationId: sagaId },
      data: { motivo: "Fondos insuficientes" },
    });

    // Debe emitir comandos de compensación en orden inverso
    expect(comandosCompensacion).toHaveLength(2);
    expect(comandosCompensacion[0].type).toBe("comando.liberar_inventario");
    expect(comandosCompensacion[1].type).toBe("comando.cancelar_orden");

    const estado = await saga.obtenerEstado(sagaId);
    expect(estado.pasoActual).toBe("fallida");
  });

  test("saga es idempotente: iniciar dos veces no ejecuta dos veces", async () => {
    await saga.iniciar({ clienteId: "cli-1", items: [] });
    
    // Segunda iniciación con los mismos datos debería ser ignorada
    await expect(
      saga.iniciar({ clienteId: "cli-1", items: [] })
    ).resolves.not.toThrow();
  });
});
```

### In-Memory Event Bus para Tests

```typescript
class InMemoryEventBus implements EventBus {
  private handlers = new Map<string, EventHandler[]>();

  async publicar<T extends DomainEvent>(evento: T): Promise<void> {
    const handlers = this.handlers.get(evento.type) ?? [];

    // Ejecutar handlers secuencialmente (determinista para tests)
    for (const handler of handlers) {
      await handler(evento);
    }
  }

  async suscribir<T extends DomainEvent>(
    eventType: string,
    handler: EventHandler<T>,
  ): Promise<void> {
    const existing = this.handlers.get(eventType) ?? [];
    existing.push(handler as EventHandler);
    this.handlers.set(eventType, existing);
  }

  // Utilidad para tests: limpiar entre tests
  clear(): void {
    this.handlers.clear();
  }
}
```

---

## 15.6 Simulación de Fallos y Chaos Engineering

```typescript
describe("Resiliencia: fallos de red y timeouts", () => {
  test("handler reintenta cuando EventBus falla temporalmente", async () => {
    const eventBusFalible = new FailingEventBus(eventBus, {
      failRate: 0.5,       // 50% de fallos
      maxRetries: 3,
    });

    const handler = new PagosHandler(procesador, eventBusFalible);

    await handler.onOrdenCreada(evento);

    // El handler debe haber reintentado y eventualmente publicado
    expect(publicacionesExitosas).toBeGreaterThan(0);
  });

  test("circuit breaker se abre tras múltiples fallos", async () => {
    const circuitBreaker = new CircuitBreaker({
      failureThreshold: 3,
      recoveryTimeout: 5000,
    });

    // Forzar 3 fallos consecutivos
    for (let i = 0; i < 3; i++) {
      await expect(
        circuitBreaker.execute(() => procesador.cobrar(datos))
      ).rejects.toThrow();
    }

    // El circuit breaker debe estar abierto
    await expect(
      circuitBreaker.execute(() => procesador.cobrar(datos))
    ).rejects.toThrow("Circuit breaker is OPEN");
  });
});
```

---

## Resumen del Capítulo

- **Unit Tests**: testean handlers y aggregates de forma aislada con mocks. Rápidos, deterministas.
- **Integration Tests**: usan Testcontainers para levantar Kafka/PostgreSQL real. Validan la integración entre componentes.
- **Consumer-Driven Contract Tests**: el consumidor define qué espera del evento. El productor verifica que cumple el contrato.
- **Saga Tests**: validan el flujo completo de orquestación, incluyendo compensación. Event Bus en memoria para tests deterministas.
- **Failure Tests**: simulan fallos de red, timeouts, y circuit breakers para validar la resiliencia del sistema.
- La **pirámide de testing EDA** añade contract tests entre integration y E2E, ya que la interfaz entre servicios es el contrato de eventos.

En el siguiente capítulo exploramos observabilidad y monitoreo en sistemas event-driven.
