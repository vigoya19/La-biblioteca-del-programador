# Capitulo 3: Topologias y Patrones de Comunicacion

La decision mas importante en EDA no es que broker usar, sino como organizar la comunicacion entre servicios. Dos estilos fundamentales: **coreografia** (servicios inteligentes, tuberias tontas) y **orquestacion** (orquestador inteligente, servicios tontos).

> 🎼 **La Analogía de la Sesión de Jazz vs. El Director de Orquesta**
> 
> Hay dos formas de lograr que múltiples músicos (que representan a nuestros microservicios) toquen una hermosa canción sin chocar ni desafinar entre sí:
> 
> 1. **Coreografía (Sesión de Improvisación de Jazz)**: No hay un líder en el escenario. Cada músico escucha con atención lo que toca su compañero y reacciona de forma totalmente autónoma.
>    - *El pianista toca un acorde alegre (Evento: `orden.creada`)*.
>    - *El bajista lo escucha y decide tocar una línea de bajo suave que combine (Evento: `pago.procesado`)*.
>    - **Ventaja**: Si el pianista se resfría y deja de tocar, el bajista y el baterista pueden seguir tocando. Es altamente flexible y desacoplado.
> 
> 2. **Orquestación (Director de Orquesta clásico)**: Hay un director en el podio con una batuta. Él decide exactamente quién toca, con qué volumen y en qué milisegundo, siguiendo una partitura estricta.
>    - *El director señala al pianista: "¡Toca!"*
>    - *El director espera a que suene la nota, la registra en su libreta y luego señala al bajista: "¡Ahora te toca a ti!"*
>    - *Si el bajista comete un error, el director tiene un plan alternativo de rescate escrito en su partitura (Mecanismo de Compensación)*.
>    - **Ventaja**: El director sabe en todo momento exactamente en qué compás de la sinfonía se encuentra la canción completa.

---


## 3.1 Coreografia

Cada servicio sabe a que eventos reaccionar y que eventos emitir. No hay un coordinador central.

```
Servicio Ordenes          Servicio Pagos           Servicio Envios
      │                        │                        │
      ├─ OrdenCreada ─────────▶│                        │
      │                        ├─ PagoProcesado ───────▶│
      │                        │                        ├─ EnvioProgramado ──▶
      │                        │                        │
      │                        │                        │
      ◀────── Ningun servicio conoce a los demas ──────▶
```

### Ejemplo: Saga de coreografia

```typescript
// Servicio Ordenes
async function crearOrden(datos: CrearOrdenDTO): Promise<void> {
  const orden = await db.orden.create({ data: datos });
  await eventBus.emit({
    type: "orden.creada",
    data: { ordenId: orden.id, clienteId: datos.clienteId, items: datos.items },
  });
}

// Servicio Pagos (reacciona a OrdenCreada)
eventBus.on("orden.creada", async (evento) => {
  const pago = await procesarPago(evento.data.clienteId, calcularTotal(evento.data.items));
  await eventBus.emit({
    type: pago.exitoso ? "pago.procesado" : "pago.rechazado",
    data: { ordenId: evento.data.ordenId, pagoId: pago.id },
  });
});

// Servicio Inventario (reacciona a PagoProcesado)
eventBus.on("pago.procesado", async (evento) => {
  await reservarInventario(evento.data.ordenId);
  await eventBus.emit({
    type: "inventario.reservado",
    data: { ordenId: evento.data.ordenId },
  });
});
```

### Ventajas y desventajas de coreografia

```typescript
// ✅ VENTAJAS
// - Bajo acoplamiento: servicios no se conocen
// - Alta resiliencia: fallo en un servicio no bloquea a otros
// - Escalabilidad: cada servicio escala segun su carga de eventos
// - Extension simple: nuevo servicio solo escucha eventos

// ❌ DESVENTAJAS
// - Logica de negocio distribuida: dificil ver el flujo completo
// - Depuracion compleja: ¿donde se atoro la orden #456?
// - Dependencias ciclicas sutiles: A emite → B emite → A emite...
// - Sin visibilidad central: ¿cuantas ordenes estan "en proceso"?
```

---

## 3.2 Orquestacion

Un **orquestador** central coordina el flujo. Los servicios ejecutan tareas especificas y reportan resultado.

```
               ┌─────────────────────┐
               │    Orquestador       │
               │  (ProcesarOrdenSaga) │
               └──┬──────┬──────┬────┘
                  │      │      │
        ┌─────────▼┐ ┌───▼───┐ ┌▼─────────┐
        │  Ordenes  │ │ Pagos │ │  Envios  │
        │ (Crear)   │ │(Cobrar)│ │(Programar)│
        └───────────┘ └───────┘ └──────────┘

El orquestador envia COMANDOS y escucha EVENTOS de respuesta.
```

### Ejemplo: Saga orquestada con Estado

```typescript
type EstadoSaga =
  | "iniciada"
  | "orden_creada"
  | "pago_procesado"
  | "envio_programado"
  | "completada"
  | "fallida";

interface SagaProcesarOrden {
  sagaId: string;
  estado: EstadoSaga;
  datos: {
    ordenId?: string;
    pagoId?: string;
    tracking?: string;
  };
}

class ProcesarOrdenSaga {
  private sagas = new Map<string, SagaProcesarOrden>();

  async iniciar(datos: CrearOrdenDTO): Promise<string> {
    const sagaId = crypto.randomUUID();
    this.sagas.set(sagaId, { sagaId, estado: "iniciada", datos: {} });

    // Paso 1: Crear orden
    await comandos.emit("crear_orden", { sagaId, ...datos });
    return sagaId;
  }

  @on("orden.creada")
  async onOrdenCreada(evento: OrdenCreadaEvent) {
    const saga = this.sagas.get(evento.metadata.sagaId)!;
    saga.estado = "orden_creada";
    saga.datos.ordenId = evento.data.ordenId;

    // Paso 2: Procesar pago
    await comandos.emit("procesar_pago", { sagaId: saga.sagaId, ordenId: saga.datos.ordenId });
  }

  @on("pago.procesado")
  async onPagoProcesado(evento: PagoProcesadoEvent) {
    const saga = this.sagas.get(evento.metadata.sagaId)!;
    saga.estado = "pago_procesado";

    // Paso 3: Programar envio
    await comandos.emit("programar_envio", { sagaId: saga.sagaId, ordenId: saga.datos.ordenId! });
  }

  @on("pago.rechazado")
  async onPagoRechazado(evento: PagoRechazadoEvent) {
    // Compensacion: cancelar orden
    const saga = this.sagas.get(evento.metadata.sagaId)!;
    saga.estado = "fallida";
    await comandos.emit("cancelar_orden", { sagaId: saga.sagaId, ordenId: saga.datos.ordenId! });
  }
}
```

### Orquestacion con AWS Step Functions

> [!TIP]
> **El Director Digital de la Orquesta**:
> En la nube de AWS, **AWS Step Functions** actúa como el director de orquesta robótico de nuestra aplicación. En lugar de escribir código TypeScript complejo para coordinar cada paso, le damos a Step Functions una "partitura" en formato **JSON** (llamada Amazon States Language). Este archivo le indica al servicio exactamente qué función ejecutar, cómo reaccionar si una tarea falla, y cuándo dar por completada la sinfonía de negocio de forma 100% automatizada.

```json
{

  "Comment": "Saga de procesar orden",
  "StartAt": "CrearOrden",
  "States": {
    "CrearOrden": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:...:function:crearOrden",
      "Next": "ProcesarPago"
    },
    "ProcesarPago": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:...:function:procesarPago",
      "Next": "VerificarPago"
    },
    "VerificarPago": {
      "Type": "Choice",
      "Choices": [
        {
          "Variable": "$.pagoExitoso",
          "BooleanEquals": true,
          "Next": "ProgramarEnvio"
        }
      ],
      "Default": "CancelarOrden"
    },
    "ProgramarEnvio": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:...:function:programarEnvio",
      "End": true
    },
    "CancelarOrden": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:...:function:cancelarOrden",
      "End": true
    }
  }
}
```

### Ventajas de orquestacion

```typescript
// ✅ Visibilidad central: un lugar para ver el flujo completo
// ✅ Manejo de errores centralizado: compensacion en un solo lugar
// ✅ Facil de razonar: logica de negocio en el orquestador
// ✅ Auditoria: estado de cada saga persiste en BD

// ❌ Acoplamiento al orquestador
// ❌ Single point of failure (mitigado con persistencia + reinicio)
// ❌ Orquestador puede ser cuello de botella
```

---

## 3.3 Event Collaboration

Los servicios colaboran compartiendo eventos de estado. Cada servicio mantiene su propia copia de los datos que necesita.

```typescript
// Servicio A: Ordenes
// Mantiene: ordenes, items
eventBus.emit({ type: "orden.creada", data: ordenCompleta });

// Servicio B: Facturacion
// Escucha OrdenCreada → guarda datos locales que necesita
eventBus.on("orden.creada", async (evento) => {
  await db.facturacion.guardarDatosOrden({
    ordenId: evento.data.ordenId,
    clienteId: evento.data.clienteId,
    total: evento.data.total,
    // Solo los campos que facturacion necesita
  });
});

// Ahora facturacion puede generar facturas SIN llamar a ordenes
// Datos locales = autonomia = resiliencia
```

---

## 3.4 Competing Consumers y Fan-Out

### Competing Consumers (escalado horizontal)

```
           ┌─── [COLA: ordenes.procesar] ───┐
           │         │           │          │
      Consumidor 1  Cons 2    Cons 3    Cons 4
      (particion 0) (part 1)  (part 2)  (part 3)

Kafka: cada particion = un consumidor activo
SQS: mensajes distribuidos entre consumidores
```

```typescript
// Escalar consumidores horizontalmente
// Kafka: max 1 consumidor por particion en el mismo grupo
// Si tienes 4 particiones, maximo 4 consumidores activos en paralelo
// Consumidores extra quedan idle como failover
```

### Fan-Out (un evento, multiples acciones)

```typescript
// Evento: OrdenConfirmada
// Fan-out a 4 servicios diferentes

eventBus.on("orden.confirmada", [
  servicioFacturacion.generarFactura,    // Accion 1
  servicioInventario.reservarProductos,   // Accion 2
  servicioEmail.enviarConfirmacion,       // Accion 3
  servicioAnalytics.registrarVenta,       // Accion 4
]);

// En AWS: SNS Topic → 4 SQS Queues (fan-out pattern)
// En Kafka: 4 consumer groups en el mismo topic
```

---

## 3.5 Choreography vs Orchestration: Matriz de Decision

| Criterio | Usa Coreografia si... | Usa Orquestacion si... |
|----------|----------------------|------------------------|
| **Complejidad del flujo** | Simple, pocos pasos (2-4) | Complejo, muchos pasos (5+) |
| **Visibilidad** | No necesitas saber estado del flujo | Necesitas saber exactamente donde esta |
| **Manejo de errores** | Simple (retry o compensacion basica) | Complejo (multi-step compensation) |
| **Tamaño del equipo** | Equipos autonomos (cada team su servicio) | Equipo central de plataforma |
| **Cambio frecuente del flujo** | Poco frecuente | Flujo cambia seguido |
| **Ejemplo** | "Cuando se crea orden → notificar + analytics" | "Procesar orden: validar → pagar → reservar → enviar → facturar" |

---

## Resumen del Capitulo

- **Coreografia**: servicios inteligentes reaccionan a eventos. Descentralizado, bajo acoplamiento, dificil de depurar.
- **Orquestacion**: orquestador coordina el flujo. Centralizado, visible, facil de razonar, acoplamiento al orquestador.
- **Event Collaboration**: cada servicio guarda los datos que necesita localmente escuchando eventos.
- **Competing Consumers** escalan horizontalmente. **Fan-out** distribuye un evento a multiples acciones.
- La regla practica: empieza con coreografia, migra a orquestacion cuando el flujo sea muy complejo para razonar distribuido.

En el siguiente capitulo nos sumergimos en Apache Kafka en profundidad.

---

← [Capítulo anterior](02-eventos-comandos-mensajes.md) | [Inicio](README.md) | [Capítulo siguiente →](04-apache-kafka.md)
