# Capítulo 11: Patrones de Datos — CQRS y Event Sourcing

> "No hay decisión arquitectónica más impactante que cómo modelas y persistes tus datos."

> [!TIP]
> Dado que **CQRS** y **Event Sourcing** son fundamentales para sistemas orientados a eventos de alta escala, te recomendamos complementar esta lectura con los capítulos prácticos de la [Arquitectura Orientada a Eventos: Guía Completa](../event-driven-architecture-programming-book/00-indice.md) en tu workspace.

## 11.1 CQRS (Command Query Responsibility Segregation)

> "Separa las operaciones de lectura de las de escritura."

En una arquitectura tradicional, usas el mismo modelo para leer y escribir. CQRS propone dos modelos distintos optimizados para cada caso.

```
Arquitectura Tradicional:
┌──────────────────────┐
│  Mismo Modelo        │
│  PedidoRepository    │
│                      │
│  findById()          │  ← Lectura
│  save()              │  ← Escritura
│  findByCliente()     │  ← Lectura compleja
└──────────────────────┘

CQRS:
┌─────────────────┐     ┌──────────────────┐
│  Command Model  │     │   Query Model    │
│  (Escritura)    │     │   (Lectura)      │
│                 │     │                  │
│  Validación     │     │  Vistas          │
│  Invariantes    │     │  precalculadas   │
│  Eventos        │────►│  Desnormalizado  │
│  PostgreSQL     │     │  Elasticsearch   │
│  (normalizado)  │     │  Redis           │
└─────────────────┘     └──────────────────┘
```

### Cuándo Usar CQRS

- **Disparidad de carga**: 1000 lecturas por cada escritura.
- **Modelos de lectura complejos**: Joins, agregaciones, múltiples fuentes.
- **Diferentes patrones de acceso**: Escrituras simples, lecturas con full-text search.
- **Escalabilidad independiente**: Escalar lecturas sin tocar escrituras.

### Cuándo NO Usar CQRS
- CRUD simple: una tabla, pocas queries.
- Baja discrepancia entre lecturas y escrituras.
- Equipos pequeños que no pueden mantener dos modelos.

### Sincronización entre Modelos

```
[Command Service] ──► EventBus ──► [Query Service]
       │                                │
       ▼                                ▼
  PostgreSQL                        Elasticsearch
  (fuente de verdad)                (optimizado para búsqueda)
```

El Query Service escucha eventos del Command Service y actualiza su modelo desnormalizado.

## 11.2 Event Sourcing

> "El estado actual es una función de todos los eventos pasados."

En lugar de guardar el estado actual, guardas la secuencia de eventos que llevaron a ese estado.

```
Modelo Tradicional (State-Oriented):
┌─────────────────────┐
│ Pedido #123          │
│ estado: "CONFIRMADO" │
│ total: 150.00        │
│ items: [...]         │
└─────────────────────┘
(Solo ves el estado final)

Event Sourcing:
┌──────────────────────────────────────┐
│ Event Stream: pedido-123              │
│                                       │
│ 1. PedidoCreado { items: [...] }     │
│ 2. ItemAñadido { producto: X, qty: 2}│
│ 3. ItemEliminado { producto: Y }     │
│ 4. PedidoConfirmado {}               │
│ 5. PagoRegistrado { monto: 150.00 }  │
└──────────────────────────────────────┘
(Guardas el historial completo)
```

### Ventajas
- **Auditoría completa**: Cada cambio está registrado.
- **Depuración temporal**: Reconstruir el estado en cualquier punto del tiempo.
- **Nuevas proyecciones**: Crear nuevas vistas reprocesando eventos históricos.
- **Analytics**: Datos crudos para análisis de comportamiento.

### Desventajas
- **Complejidad**: Diseñar eventos, versionarlos, manejar migraciones.
- **Consultas**: No puedes hacer `SELECT * WHERE estado = 'CONFIRMADO'` sin proyecciones.
- **Eventual consistency**: Las proyecciones van con retraso.
- **Tamaño**: El event store crece indefinidamente.

### Snapshots
Para no reprocesar millones de eventos cada vez:

```
Evento 1, 2, 3, ..., 500 → Snapshot (estado en evento 500)
Evento 501, 502, ..., 1000 → Snapshot (estado en evento 1000)

Al reconstruir: cargar último snapshot + eventos posteriores.
```

## 11.3 CQRS + Event Sourcing (La Combinación Perfecta)

```
┌──────────────┐     Eventos      ┌────────────────┐
│ Command Side │─────────────────►│  Query Side    │
│              │                  │                │
│ Write Model  │                  │ Read Models    │
│ Event Store  │                  │ Proyecciones   │
└──────────────┘                  └────────────────┘
```

Esta combinación es el patrón por excelencia para sistemas con:
- Auditoría requerida (financiero, salud).
- Reglas de negocio complejas basadas en historial.
- Múltiples representaciones de los mismos datos.

## 11.4 Saga Pattern

En microservicios no tienes transacciones ACID distribuidas. La Saga coordina una transacción de negocio que abarca múltiples servicios mediante una secuencia de transacciones locales.

### Choreography (Coreografía)
Cada servicio publica eventos y reacciona a eventos de otros. Descentralizado.

```
Servicio A ──► Evento: PedidoCreado ──► Servicio B
                                              │
                                              ▼
Servicio D ◄── Evento: PagoFallido ◄── Servicio C
   │
   ▼ (compensa)
```

### Orchestration (Orquestación)
Un orquestador central coordina los pasos.

```
          ┌──────────────┐
          │  Orquestador │
          │    (Saga)    │
          └──────┬───────┘
     ┌───────────┼───────────┐
     ▼           ▼           ▼
Servicio A  Servicio B  Servicio C
```

### Compensaciones
Si un paso falla, debes deshacer los pasos anteriores:

```
1. Crear Pedido     ✓
2. Reservar Stock   ✓
3. Cobrar Pago      ✗ (falla)
4. Liberar Stock    ← compensación paso 2
5. Cancelar Pedido  ← compensación paso 1
```

---

> **Reflexión del capítulo**: CQRS y Event Sourcing son poderosos, pero no son la respuesta para todo. Si los adoptas sin necesidad real, añadirás una complejidad que te perseguirá durante años. Evalúa el costo antes de adoptarlos.

---

← [Capítulo anterior](10-mensajeria-eventos.md) | [Inicio](README.md) | [Capítulo siguiente →](12-resiliencia.md)
