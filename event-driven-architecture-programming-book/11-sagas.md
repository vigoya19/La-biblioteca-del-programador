# Capítulo 11: Sagas y Process Managers

En una arquitectura distribuida orientada a eventos, no existe una transacción ACID que abarque múltiples servicios. Las Sagas resuelven este problema mediante una secuencia de transacciones locales, cada una con su paso de compensación en caso de fallo.

> "Una saga es un conjunto de transacciones locales. Si una falla, compensas las anteriores. No hay rollback global: hay compensación." — Pat Helland

---

## 11.1 El Problema de las Transacciones Distribuidas

```
SIN SAGA: Transacción distribuida (Two-Phase Commit)
─────────────────────────────────────────────────────
  Servicio A ──prepare──▶ Servicio B ──prepare──▶ Servicio C
     │                       │                       │
     └── commit/rollback ────┴── commit/rollback ────┘
     
  Problemas:
  ❌ Bloqueos distribuidos (lento, frágil)
  ❌ No escala (coordinador es cuello de botella)
  ❌ Un nodo caído bloquea toda la transacción
  ❌ No soportado por la mayoría de tecnologías cloud

CON SAGA: Compensación local
────────────────────────────
  Paso 1: CrearOrden         ✅ Éxito
  Paso 2: ReservarInventario ✅ Éxito
  Paso 3: ProcesarPago       ❌ Falla
  ──▶ Compensar Paso 2: LiberarInventario
  ──▶ Compensar Paso 1: CancelarOrden
```

---

## 11.2 Saga: Coreografía vs Orquestación

### Saga Coreografiada (Event-Driven)

Cada servicio reacciona a eventos y decide qué hacer. No hay coordinador central.

```
Servicio A           Servicio B           Servicio C
(Órdenes)            (Inventario)         (Pagos)
    │                     │                    │
    ├─ OrdenCreada ──────▶│                    │
    │                     ├─ InvReservado ────▶│
    │                     │                    ├─ PagoProcesado ──▶
    │                     │                    │
    │                     │   Si falla pago:   │
    │                     ◀── PagoRechazado ───┤
    │                     ├─ LiberarInv        │
    ◀────── InvLiberado ──┘                    │
    ├─ CancelarOrden                           │
```

```typescript
// ─── Saga de coreografía: cada servicio es autónomo ───

// Servicio Órdenes
class OrdenesService {
  async crearOrden(dto: CrearOrdenDTO): Promise<void> {
    const orden = await this.repositorio.crear({
      id: crypto.randomUUID(),
      clienteId: dto.clienteId,
      items: dto.items,
      estado: "pendiente",
    });

    await this.eventBus.publicar({
      type: "orden.creada",
      data: { ordenId: orden.id, items: dto.items, total: dto.total },
    });
  }

  // Reaccionar a fallos
  @onEvent("pago.rechazado")
  async onPagoRechazado(evento: PagoRechazado): Promise<void> {
    await this.repositorio.actualizarEstado(evento.data.ordenId, "cancelada");
    // No necesitamos compensar nada más: inventario se libera por su cuenta
  }
}

// Servicio Inventario
class InventarioService {
  @onEvent("orden.creada")
  async onOrdenCreada(evento: OrdenCreada): Promise<void> {
    try {
      await this.reservarItems(evento.data.items);
      await this.eventBus.publicar({
        type: "inventario.reservado",
        data: { ordenId: evento.data.ordenId, items: evento.data.items },
      });
    } catch (error) {
      await this.eventBus.publicar({
        type: "inventario.no_disponible",
        data: { ordenId: evento.data.ordenId, motivo: error.message },
      });
    }
  }

  @onEvent("pago.rechazado")
  async onPagoRechazado(evento: PagoRechazado): Promise<void> {
    // Compensación automática
    await this.liberarReserva(evento.data.ordenId);
    await this.eventBus.publicar({
      type: "inventario.liberado",
      data: { ordenId: evento.data.ordenId },
    });
  }
}
```

**Ventajas de la coreografía:**
- Bajo acoplamiento: servicios no se conocen
- Resiliencia: sin punto único de fallo
- Simple de extender: nuevo servicio solo escucha eventos

**Desventajas:**
- Lógica de saga dispersa: difícil ver el flujo completo
- Dependencias cíclicas sutiles
- Sin visibilidad del estado de la saga

### Saga Orquestada (Command-Driven)

Un **orquestador** central coordina todos los pasos. Los servicios ejecutan comandos y reportan resultados.

```
                        ┌──────────────────┐
                        │    Orquestador    │
                        │  ProcesarOrden    │
                        │     Saga          │
                        └──┬──────┬──────┬─┘
                           │      │      │
                  ┌────────▼┐ ┌───▼───┐ ┌▼────────┐
                  │ Crear    │ │Reservar│ │Procesar │
                  │ Orden    │ │Invent  │ │Pago     │
                  └──────────┘ └───────┘ └─────────┘
```

```typescript
// ─── Saga Orquestada ───

type Paso = "orden_creada" | "inventario_reservado" | "pago_procesado" | "completada" | "fallida";

interface SagaProcesarOrden {
  sagaId: string;
  ordenId: string;
  pasoActual: Paso;
  datos: {
    clienteId: string;
    items: ItemDTO[];
    total: number;
    pagoId?: string;
    error?: string;
  };
  historial: { paso: string; timestamp: string; exitoso: boolean }[];
}

class ProcesarOrdenSagaOrquestador {
  private sagas = new Map<string, SagaProcesarOrden>();

  constructor(
    private eventBus: EventBus,
    private db: Database,
  ) {
    // Escuchar eventos de respuesta de los servicios
    this.eventBus.on("orden.creada", this.onOrdenCreada.bind(this));
    this.eventBus.on("inventario.reservado", this.onInventarioReservado.bind(this));
    this.eventBus.on("inventario.no_disponible", this.onInventarioNoDisponible.bind(this));
    this.eventBus.on("pago.procesado", this.onPagoProcesado.bind(this));
    this.eventBus.on("pago.rechazado", this.onPagoRechazado.bind(this));
  }

  async iniciar(dto: CrearOrdenDTO): Promise<string> {
    const sagaId = crypto.randomUUID();
    const ordenId = crypto.randomUUID();

    const saga: SagaProcesarOrden = {
      sagaId,
      ordenId,
      pasoActual: "orden_creada",
      datos: { ...dto },
      historial: [],
    };

    this.sagas.set(sagaId, saga);
    await this.persistirSaga(saga);

    // ─── Paso 1: Crear Orden ───
    await this.eventBus.publicar({
      type: "comando.crear_orden",
      data: { sagaId, ordenId, clienteId: dto.clienteId, items: dto.items, total: dto.total },
    });

    logger.info("Saga iniciada", { sagaId, ordenId });
    return sagaId;
  }

  // ─── Handlers de eventos de respuesta ───

  private async onOrdenCreada(evento: OrdenCreada): Promise<void> {
    const saga = await this.obtenerSaga(evento.metadata.causationId);
    if (!saga) return;

    saga.pasoActual = "orden_creada";
    this.registrarPaso(saga, "orden_creada", true);

    // ─── Paso 2: Reservar Inventario ───
    await this.eventBus.publicar({
      type: "comando.reservar_inventario",
      data: { sagaId: saga.sagaId, ordenId: saga.ordenId, items: saga.datos.items },
    });
  }

  private async onInventarioReservado(evento: InventarioReservado): Promise<void> {
    const saga = await this.obtenerSaga(evento.metadata.causationId);
    if (!saga) return;

    saga.pasoActual = "inventario_reservado";
    this.registrarPaso(saga, "inventario_reservado", true);

    // ─── Paso 3: Procesar Pago ───
    await this.eventBus.publicar({
      type: "comando.procesar_pago",
      data: { sagaId: saga.sagaId, ordenId: saga.ordenId, monto: saga.datos.total, clienteId: saga.datos.clienteId },
    });
  }

  private async onPagoProcesado(evento: PagoProcesado): Promise<void> {
    const saga = await this.obtenerSaga(evento.metadata.causationId);
    if (!saga) return;

    saga.pasoActual = "completada";
    this.registrarPaso(saga, "pago_procesado", true);
    saga.datos.pagoId = evento.data.pagoId;

    logger.info("Saga completada exitosamente", { sagaId: saga.sagaId, ordenId: saga.ordenId });
  }

  // ─── Compensación ───

  private async onPagoRechazado(evento: PagoRechazado): Promise<void> {
    const saga = await this.obtenerSaga(evento.metadata.causationId);
    if (!saga) return;

    this.registrarPaso(saga, "pago_rechazado", false);
    await this.compensar(saga, evento.data.motivo);
  }

  private async onInventarioNoDisponible(evento: InventarioNoDisponible): Promise<void> {
    const saga = await this.obtenerSaga(evento.metadata.causationId);
    if (!saga) return;

    this.registrarPaso(saga, "inventario_no_disponible", false);
    await this.compensar(saga, evento.data.motivo);
  }

  // ─── Lógica de compensación ───
  private async compensar(saga: SagaProcesarOrden, motivo: string): Promise<void> {
    saga.pasoActual = "fallida";
    saga.datos.error = motivo;

    const pasos = saga.historial.filter(h => h.exitoso).map(h => h.paso);

    // Compensar en orden INVERSO
    for (const paso of pasos.reverse()) {
      switch (paso) {
        case "inventario_reservado":
          await this.eventBus.publicar({
            type: "comando.liberar_inventario",
            data: { sagaId: saga.sagaId, ordenId: saga.ordenId, items: saga.datos.items },
          });
          break;

        case "orden_creada":
          await this.eventBus.publicar({
            type: "comando.cancelar_orden",
            data: { sagaId: saga.sagaId, ordenId: saga.ordenId, motivo },
          });
          break;

        case "pago_procesado":
          // ¡El pago ya se procesó! Compensar = reembolso
          await this.eventBus.publicar({
            type: "comando.reembolsar_pago",
            data: { sagaId: saga.sagaId, ordenId: saga.ordenId, pagoId: saga.datos.pagoId!, motivo },
          });
          break;
      }
    }

    await this.persistirSaga(saga);
    logger.error("Saga fallida y compensada", { sagaId: saga.sagaId, motivo });
  }

  // ─── Persistencia ───
  private async persistirSaga(saga: SagaProcesarOrden): Promise<void> {
    await this.db.query(
      `INSERT INTO sagas (saga_id, orden_id, paso_actual, datos, historial)
       VALUES ($1, $2, $3, $4, $5)
       ON CONFLICT (saga_id) DO UPDATE
       SET paso_actual = $3, datos = $4, historial = $5, actualizada_en = NOW()`,
      [saga.sagaId, saga.ordenId, saga.pasoActual, JSON.stringify(saga.datos), JSON.stringify(saga.historial)],
    );
  }

  private registrarPaso(saga: SagaProcesarOrden, paso: string, exitoso: boolean): void {
    saga.historial.push({ paso, timestamp: new Date().toISOString(), exitoso });
  }
}
```

---

## 11.3 Saga con AWS Step Functions

Step Functions es la implementación managed de saga orquestada en AWS:

```json
{
  "Comment": "Saga: Procesar Orden",
  "StartAt": "CrearOrden",
  "States": {
    "CrearOrden": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:...:crearOrden",
      "Next": "ReservarInventario",
      "Catch": [{ "ErrorEquals": ["States.ALL"], "Next": "SagaFallida" }]
    },
    "ReservarInventario": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:...:reservarInventario",
      "Next": "ProcesarPago",
      "Catch": [{
        "ErrorEquals": ["SinStockError"],
        "Next": "CompensarCreacionOrden"
      }]
    },
    "ProcesarPago": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:...:procesarPago",
      "Next": "VerificarPago",
      "Catch": [{
        "ErrorEquals": ["PagoRechazadoError"],
        "ResultPath": "$.errorInfo",
        "Next": "CompensarReservaInventario"
      }]
    },
    "VerificarPago": {
      "Type": "Choice",
      "Choices": [
        {
          "Variable": "$.pagoExitoso",
          "BooleanEquals": true,
          "Next": "SagaExitosa"
        }
      ],
      "Default": "CompensarReservaInventario"
    },

    "── Compensaciones ──": "───",

    "CompensarCreacionOrden": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:...:cancelarOrden",
      "Next": "SagaFallida"
    },
    "CompensarReservaInventario": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:...:liberarInventario",
      "Next": "CompensarCreacionOrden"
    },
    "SagaFallida": {
      "Type": "Fail",
      "Error": "SagaFallida",
      "Cause": "La saga no pudo completarse"
    },
    "SagaExitosa": {
      "Type": "Succeed"
    }
  }
}
```

---

## 11.4 Process Manager como Alternativa a la Saga

Un Process Manager es más inteligente que una saga: mantiene estado, maneja eventos arbitrarios (no solo pasos secuenciales), y puede tomar decisiones complejas.

```
SAGA: "Secuencia A → B → C"
  Cada paso es un comando lineal. Si falla, compensa hacia atrás.

PROCESS MANAGER: "Máquina de estados reactiva"
  Espera N eventos, toma decisiones según estado,
  dispara comandos cuando se cumplen condiciones.
```

```typescript
// ─── Process Manager: Verificación de Identidad ───
// No es lineal. Depende de múltiples eventos externos.

type EstadoVerificacion = "pendiente" | "documentos_recibidos" | "selfie_recibida"
  | "verificacion_externa_completada" | "aprobada" | "rechazada" | "expirada";

interface ProcesoVerificacionIdentidad {
  procesoId: string;
  usuarioId: string;
  estado: EstadoVerificacion;
  tiempoLimite: Date;
  documentos: string[];
  resultadoVerificacionExterna?: "match" | "no_match";
}

class VerificacionIdentidadProcessManager {
  @onEvent("usuario.registrado")
  async onUsuarioRegistrado(evento: UsuarioRegistrado): Promise<void> {
    const proceso: ProcesoVerificacionIdentidad = {
      procesoId: crypto.randomUUID(),
      usuarioId: evento.data.usuarioId,
      estado: "pendiente",
      tiempoLimite: new Date(Date.now() + 24 * 60 * 60 * 1000), // 24 horas
      documentos: [],
    };

    await this.guardarProceso(proceso);

    // Solicitar documentos
    await this.comandos.enviar({
      type: "solicitar_documentos",
      data: { usuarioId: evento.data.usuarioId },
    });
  }

  @onEvent("documento.subido")
  async onDocumentoSubido(evento: DocumentoSubido): Promise<void> {
    const proceso = await this.obtenerProceso(evento.data.usuarioId);
    if (!proceso || proceso.estado === "expirada") return;

    proceso.documentos.push(evento.data.documentoId);

    // Verificar si ya recibimos los dos documentos requeridos
    if (proceso.documentos.length >= 2) {
      proceso.estado = "documentos_recibidos";

      // Disparar verificación externa
      await this.comandos.enviar({
        type: "verificar_identidad_externa",
        data: { usuarioId: proceso.usuarioId, documentos: proceso.documentos },
      });
    }

    await this.guardarProceso(proceso);
  }

  @onEvent("verificacion.externa.completada")
  async onVerificacionExterna(evento: VerificacionExterna): Promise<void> {
    const proceso = await this.obtenerProceso(evento.data.usuarioId);
    if (!proceso) return;

    proceso.estado = "verificacion_externa_completada";
    proceso.resultadoVerificacionExterna = evento.data.resultado;

    if (evento.data.resultado === "match") {
      proceso.estado = "aprobada";
      await this.comandos.enviar({
        type: "activar_cuenta",
        data: { usuarioId: proceso.usuarioId },
      });
      await this.eventBus.publicar({
        type: "verificacion.aprobada",
        data: { usuarioId: proceso.usuarioId },
      });
    } else {
      proceso.estado = "rechazada";
      await this.eventBus.publicar({
        type: "verificacion.rechazada",
        data: { usuarioId: proceso.usuarioId, motivo: "No match en verificación externa" },
      });
    }

    await this.guardarProceso(proceso);
  }

  @onTimer("cada_hora")
  async verificarExpirados(): Promise<void> {
    const expirados = await this.db.query(
      `SELECT * FROM procesos_verificacion
       WHERE estado IN ('pendiente', 'documentos_recibidos')
         AND tiempo_limite < NOW()`,
    );

    for (const p of expirados.rows) {
      p.estado = "expirada";
      await this.guardarProceso(p);

      await this.eventBus.publicar({
        type: "verificacion.expirada",
        data: { usuarioId: p.usuario_id },
      });
    }
  }
}
```

---

## 11.5 Manejo de Fallos, Reintentos y Compensación

### Idempotencia en Sagas

```typescript
// Cada comando de saga debe ser idempotente
// Usar sagaId + paso como clave de idempotencia

class IdempotentCommandHandler {
  private ejecutados = new Set<string>(); // En producción: Redis/Tabla

  async handle(comando: ComandoSaga): Promise<void> {
    const idempotencyKey = `${comando.sagaId}:${comando.tipo}`;

    if (this.ejecutados.has(idempotencyKey)) {
      logger.warn("Comando duplicado, ignorando", { idempotencyKey });
      return;
    }

    await this.ejecutarComando(comando);
    this.ejecutados.add(idempotencyKey);

    // Publicar evento de confirmación
    await this.eventBus.publicar({
      type: `${comando.tipo}.completado`,
      data: { sagaId: comando.sagaId, resultado: "exito" },
    });
  }
}
```

### Reintentos por Timeout

```typescript
class SagaConTimeout {
  private sagas = new Map<string, { timeout: NodeJS.Timeout; saga: SagaProcesarOrden }>();

  async iniciar(dto: CrearOrdenDTO): Promise<string> {
    const sagaId = crypto.randomUUID();
    // ...

    // Si la saga no se completa en 30s, disparar timeout
    const timeout = setTimeout(async () => {
      const saga = this.sagas.get(sagaId);
      if (saga && saga.saga.pasoActual !== "completada") {
        logger.error("Saga timeout", { sagaId, paso: saga.saga.pasoActual });
        await this.compensar(saga.saga, "timeout");
      }
    }, 30000);

    this.sagas.set(sagaId, { timeout, saga });
    return sagaId;
  }

  private onSagaCompletada(sagaId: string): void {
    const entry = this.sagas.get(sagaId);
    if (entry) {
      clearTimeout(entry.timeout);
      this.sagas.delete(sagaId);
    }
  }
}
```

---

## Resumen del Capítulo

- Las **Sagas** resuelven transacciones distribuidas mediante una secuencia de pasos locales con compensación.
- **Saga Coreografiada**: servicios reaccionan a eventos. Bajo acoplamiento, pero lógica dispersa.
- **Saga Orquestada**: un orquestador coordina los pasos. Visibilidad central, pero acoplamiento al orquestador.
- La **compensación** es semántica (no rollback técnico): si cobraste, reembolsas. Si reservaste, liberas.
- **AWS Step Functions** implementa sagas orquestadas como servicio gestionado.
- **Process Managers** son más flexibles que las sagas: manejan estados complejos y múltiples eventos, no solo pasos secuenciales.
- **Idempotencia** y **timeouts** son críticos: cada paso de saga debe tolerar reintentos y mensajes duplicados.

En el siguiente capítulo exploramos la consistencia eventual en detalle.

---

← [Capítulo anterior](10-cqrs.md) | [Inicio](README.md) | [Capítulo siguiente →](12-eventual-consistency.md)
