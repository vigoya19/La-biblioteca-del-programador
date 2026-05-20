# Capitulo 1: Fundamentos de la Arquitectura Orientada a Eventos

La Arquitectura Orientada a Eventos (Event-Driven Architecture, EDA) es un paradigma donde los componentes del sistema se comunican mediante la produccion, deteccion y reaccion a eventos. No es una tecnologia especifica — es un modelo mental que cambia como disenamos sistemas.

> "Un evento es algo que ocurrio en el pasado y que otros componentes pueden necesitar saber." — Martin Fowler

---

## 1.1 ¿Que es Event-Driven Architecture?

En una arquitectura tradicional **request-driven** (REST, gRPC), el flujo es sincrono: el Cliente A llama al Servicio B y espera respuesta. Si B llama a C, la latencia se acumula y A espera encadenado.

```
Request-Driven (orquestacion sincrona):
Cliente ──POST──▶ Servicio A ──GET──▶ Servicio B ──POST──▶ Servicio C
   ◀───────── 200 OK ◀──────── 200 OK ◀──────── 201 OK
   │                               │
   └── Tiempo total = A + B + C ──┘
```

En EDA, los servicios **emiten eventos** cuando algo significativo ocurre. Otros servicios **reaccionan** a esos eventos de forma asincrona:

```
Event-Driven (coreografia asincrona):
Servicio A ──▶ [Evento: OrdenCreada]
                  │
     ┌────────────┼────────────┐
     ▼            ▼            ▼
Servicio B    Servicio C    Servicio D
(Pagos)      (Inventario)  (Notificaciones)
```

### Los tres principios fundamentales

1. **Emite, no preguntes**: En vez de preguntar "¿cual es el estado de X?", escuchas el evento "X cambio a Y" cuando ocurre.

2. **Desacoplamiento temporal**: El emisor no sabe quien consume el evento, ni cuando. Puede haber cero, uno o cien consumidores.

3. **Evento como fuente de verdad**: El evento es inmutable. Ocurrio. Nunca se modifica ni se elimina. Es un hecho historico.

---

## 1.2 Eventos vs Comandos vs Consultas

Esta distincion es FUNDAMENTAL. Confundirlos es el error #1 en EDA.

| | Evento | Comando | Consulta |
|---|--------|---------|----------|
| **Proposito** | Informar que algo PASO | Solicitar que algo PASE | Preguntar el estado actual |
| **Nombrado** | Pasado: `OrdenCreada`, `PagoRechazado` | Imperativo: `CrearOrden`, `ProcesarPago` | Pregunta: `ObtenerOrden`, `BuscarUsuario` |
| **Direccion** | Emisor → Consumidores | Solicitante → Destinatario | Solicitante → Servicio |
| **Respuesta esperada** | Ninguna | Si (sync o async) | Siempre sincrona |
| **Puede ser rechazado** | No (ya ocurrio) | Si (no autorizado, invalido) | N/A |
| **Inmutabilidad** | Inmutable | Mutable (antes de ejecutar) | N/A |

```typescript
// ❌ MAL: evento nombrado como comando
{ type: "ProcesarPago", ordenId: "123", monto: 99.99 }
// "ProcesarPago" es un comando, no un evento. No ha pasado aun.

// ✅ BIEN: evento en pasado
{ type: "PagoProcesado", ordenId: "123", monto: 99.99, metodo: "tarjeta", timestamp: "..." }

// ❌ MAL: evento demasiado generico
{ type: "Actualizado", entidad: "orden", id: "123" }
// ¿Actualizado como? ¿Que cambio? Sin informacion util.

// ✅ BIEN: evento especifico y descriptivo
{ type: "OrdenEnviada", ordenId: "123", transportista: "DHL", trackingNumber: "1Z999" }
```

---

## 1.3 Paradigmas: Request-Driven vs Event-Driven

### Request-Driven (el modelo tradicional)

```
Ventajas:
✅ Simplicidad mental: request → response
✅ Facil de depurar: trazas sincronas
✅ Consistencia fuerte: todo en una transaccion
✅ Tooling maduro: REST, GraphQL, gRPC

Desventajas:
❌ Acoplamiento temporal: el cliente espera
❌ Cascada de fallos: si B falla, A falla
❌ No escala en complejidad: 20 servicios sincronos = fragil
❌ Single point of failure: un servicio lento frena todo
```

### Event-Driven

```
Ventajas:
✅ Desacoplamiento: servicios no se conocen entre si
✅ Resiliencia: si un consumidor falla, los demas siguen
✅ Escalabilidad: cada servicio escala independiente
✅ Extensibilidad: nuevo consumidor sin tocar productores
✅ Auditoria natural: eventos como log inmutable

Desventajas:
❌ Complejidad: depurar es mas dificil (trazas distribuidas)
❌ Consistencia eventual: no hay garantia inmediata
❌ Duplicacion de datos: cada servicio tiene su propia "verdad"
❌ Tooling menos maduro: necesitas broker, schema registry, monitoring
```

---

## 1.4 El Manifiesto Reactivo y EDA

El Manifiesto Reactivo (2014) define sistemas con cuatro propiedades:

```
┌─────────────────────────────────────────────────────────┐
│                   SISTEMAS REACTIVOS                     │
│                                                         │
│  RESPONSIVOS        RESILIENTES     ELASTICOS           │
│  (Responden         (Se recuperan   (Escalan            │
│   rapido)            de fallos)      bajo carga)         │
│                                                         │
│              IMPULSADOS POR MENSAJES                    │
│         (Message-Driven = Event-Driven)                  │
└─────────────────────────────────────────────────────────┘
```

EDA implementa naturalmente estos principios:

- **Responsivo**: eventos procesados asincronamente, sin bloqueo.
- **Resiliente**: cada servicio aislado. Fallo en pagos no tumba inventario.
- **Elastico**: consumidores escalan independientemente.
- **Message-Driven**: la comunicacion por eventos es el pegamento.

---

## 1.5 Topologias EDA

### Broker Topology (Choreography)

```
         ┌──────────────────────┐
         │     Event Broker      │
         │   (Kafka, RabbitMQ)   │
         └──┬──────┬──────┬─────┘
            │      │      │
    ┌───────▼┐ ┌───▼───┐ ┌▼───────┐
    │ Ordenes│ │Pagos  │ │ Envios │
    └────────┘ └───────┘ └────────┘

Los servicios se comunican SOLO via el broker.
Ningun servicio conoce a otro directamente.
```

### Mediator Topology (Orchestration)

```
    ┌─────────────────────────┐
    │       Orchestrator       │
    │  (Step Functions, Camunda│
    │   Temporal, Zeebe)       │
    └──┬───────┬──────┬──────┬─┘
       │       │      │      │
  ┌────▼──┐ ┌─▼───┐ ┌▼──┐ ┌─▼──────┐
  │Ordenes│ │Pagos│ │Inv│ │Notificac│
  └───────┘ └─────┘ └───┘ └─────────┘

El orchestrator coordina el flujo.
Los servicios son "tontos": ejecutan y responden.
```

### Comparativa

| | Broker (Coreografia) | Mediator (Orquestacion) |
|---|---|---|
| **Control de flujo** | Distribuido | Centralizado |
| **Visibilidad** | Dificil (logica dispersa) | Clara (orquestador central) |
| **Acoplamiento** | Bajo | Medio (todos conocen al orquestador) |
| **Complejidad** | Cada servicio gestiona su parte | Orquestador concentra la complejidad |
| **Ejemplo** | Microservicios Kafka | AWS Step Functions, Temporal |

---

## 1.6 Modelo de Madurez EDA

```
Nivel 1: Eventos de notificacion
  "Algo paso, por si te interesa"
  Ej: "Usuario cambio su email"

Nivel 2: Eventos con transferencia de estado
  "Algo paso y aqui tienes los datos"
  Ej: "Usuario cambio email a nuevo@test.com"

Nivel 3: Event Sourcing
  "Todo cambio es un evento. El estado se reconstruye"
  Ej: Secuencia de eventos: UsuarioCreado → EmailCambiado → EmailVerificado

Nivel 4: CQRS + Event Sourcing
  "Escrituras via eventos, lecturas via proyecciones optimizadas"
  Ej: Tabla usuarios_lectura se actualiza via proyeccion de eventos
```

---

## Resumen del Capitulo

- EDA es un paradigma donde componentes se comunican produciendo y reaccionando a eventos.
- La distincion **Evento** (pasado, inmutable) vs **Comando** (imperativo, intencion) vs **Consulta** (pregunta) es fundamental.
- EDA desacopla temporalmente productores de consumidores.
- El Manifiesto Reactivo encuentra su implementacion natural en EDA.
- Dos topologias principales: Broker (coreografia descentralizada) y Mediator (orquestacion centralizada).
- El modelo de madurez va de eventos simples de notificacion hasta Event Sourcing + CQRS.

En el siguiente capitulo profundizamos en la anatomia de eventos, comandos y patrones de mensajeria.

---

[Inicio](README.md) | [Capítulo siguiente →](02-eventos-comandos-mensajes.md)
