# Capítulo 17: Anti-Patrones y Buenas Prácticas

Después de 16 capítulos de teoría y práctica, este capítulo final destila las lecciones aprendidas por equipos que han implementado EDA en producción. Saber qué NO hacer es tan importante como saber qué hacer.

> "Aprende de los errores de otros. No vivirás lo suficiente para cometerlos todos tú mismo." — Eleanor Roosevelt

---

## 17.1 Anti-Patrones

### Anti-Patrón #1: El Evento Dios (Event God)

Emitir un solo evento genérico que contiene toda la información posible.

```typescript
// ❌ ANTI-PATRÓN: Evento Dios
{
  type: "datos.actualizados",
  data: {
    ordenId: "123",
    cliente: { /* TODO el perfil del cliente */ },
    items: [ /* TODOS los detalles de items */ ],
    pago: { /* TODA la info de pago */ },
    envio: { /* TODA la info de envío */ },
    factura: { /* TODA la factura */ },
    metadata: { /* TONELADAS de metadata */ },
  }
}

// PROBLEMAS:
// - ¿Qué cambió realmente? Imposible saber.
// - Todos los consumidores reciben datos que no necesitan.
// - Acoplamiento masivo: un cambio en "cliente" rompe a TODOS.
// - Tamaño de payload enorme: problema de performance en Kafka.

// ✅ SOLUCIÓN: Eventos pequeños y específicos
{ type: "orden.creada", data: { ordenId, clienteId, total, items } }
{ type: "pago.procesado", data: { pagoId, ordenId, monto, metodo } }
{ type: "orden.enviada", data: { ordenId, tracking } }
```

### Anti-Patrón #2: Callback Hell Asíncrono

Encadenar eventos en una secuencia rígida donde cada servicio espera al anterior.

```typescript
// ❌ ANTI-PATRÓN: Callback hell con eventos
// Servicio A emite EventoA → B emite EventoB → C emite EventoC → ...
// Si B falla, C nunca se ejecuta. Si A es lento, toda la cadena espera.

// Una orden tarda 30 segundos en procesarse porque cada paso espera al anterior.

// ✅ SOLUCIÓN: Fan-out + Coreografía paralela
// OrdenCreada → [Pagos, Notificaciones, Analytics] en paralelo
// PagoProcesado → [Inventario, Envíos, Facturación] en paralelo

// Si un paso falla, los otros siguen. Compensación solo donde sea necesario.
```

### Anti-Patrón #3: Data Overload (Sobrecarga de Datos)

Transferir todo el estado en cada evento, incluso datos que no cambiaron.

```typescript
// ❌ ANTI-PATRÓN: Cada evento contiene TODO el estado de la orden
{ type: "orden.direccion_envio_actualizada", data: {
  ordenId: "123",
  clienteId: "456",      // No cambió
  items: [...],          // No cambió
  total: 159.98,         // No cambió
  estado: "creada",      // No cambió
  direccionEnvio: { calle: "Nueva Calle 42", ... } // ← SOLO ESTO CAMBIÓ
}}

// PROBLEMA:
// - Consumidores procesan 10x más datos de los necesarios.
// - Imposible saber qué cambió sin diffing.
// - Mayor costo de almacenamiento y red.

// ✅ SOLUCIÓN: Eventos con solo el delta + referencia
{ type: "orden.direccion_envio_actualizada", data: {
  ordenId: "123",
  direccionEnvio: { calle: "Nueva Calle 42", ciudad: "Barcelona" },
  // El consumidor ya tiene el resto de los datos (Event Collaboration)
}}
```

### Anti-Patrón #4: Ghost Events (Eventos Fantasma)

Publicar eventos que nadie consume. O peor: dejar de consumir eventos sin avisar.

```typescript
// ❌ ANTI-PATRÓN: Evento que nadie consume
// "Vamos a publicar UsuarioActualizado por si acaso algún día alguien lo necesita."
// 6 meses después: 50M eventos acumulados, nadie los lee, desperdicio de recursos.

// ✅ SOLUCIÓN: Todo evento debe tener al menos un consumidor conocido.
// Si un evento no tiene consumidores → no es un evento, es un log.
// Schema Registry debe mostrar: "consumed by: facturacion, analytics"

// Para evitar ghost consumers:
// Health check que alerte si un consumer group no tiene consumidores activos.
```

### Anti-Patrón #5: Ordenamiento Global Innecesario

Forzar ordenamiento estricto donde no es necesario, creando cuellos de botella.

```typescript
// ❌ ANTI-PATRÓN: Todo en una partición para "garantizar orden"
// 1 partición en Kafka, 1 consumidor
// Throughput limitado a un solo hilo. No escala.

// ✅ SOLUCIÓN: Particionar por entidad relevante
// Partition key = orderId → eventos de la MISMA orden van en orden
// Partition key = customerId → eventos del MISMO cliente van en orden
// Particiones diferentes pueden procesarse en paralelo sin problema.

await producer.send({
  topic: "ordenes",
  messages: [
    {
      key: `orden-${ordenId}`,    // ← Particionar por orden
      value: JSON.stringify(evento),
    },
  ],
});
```

### Anti-Patrón #6: Coreografía Descontrolada

Usar coreografía para absolutamente todo, perdiendo visibilidad.

```typescript
// ❌ ANTI-PATRÓN: 15 servicios coreografíados
// "¿Por qué la orden #456 no se envió?"
// → Nadie sabe. La lógica está dispersa en 15 servicios.
// → Debugging requiere revisar logs de los 15 servicios.

// ✅ SOLUCIÓN: Coreografía para lo simple, Orquestación para lo complejo
// Regla práctica:
// - 2-4 servicios colaborando → Coreografía
// - 5+ servicios, flujo con muchas condiciones → Orquestación (Saga)
// - Necesitas visibilidad del estado → Process Manager
```

### Anti-Patrón #7: Sin Estrategia de Versionado

Cambiar schemas de eventos sin plan de compatibilidad, rompiendo consumidores en producción.

```typescript
// ❌ ANTI-PATRÓN: "Solo añado un campo, ¿qué puede salir mal?"
// Productor: añade "direccionEnvio" como campo REQUERIDO.
// Consumidor v1: intenta deserializar. Falla. Boom.

// ✅ SOLUCIÓN:
// 1. Schema Registry con FULL compatibility
// 2. Campos nuevos SIEMPRE con default value
// 3. Nunca eliminar campos requeridos sin nueva versión
// 4. Versionado dual durante migración
```

---

## 17.2 Principios de Diseño EDA

### Principio #1: Event-First Design

Diseña pensando en eventos, no en APIs o bases de datos.

```
❌ CRUD-First:
   "Necesito una tabla de órdenes con estado, y endpoints POST/GET/PUT."

✅ Event-First:
   "¿Qué OCURRE en el negocio?"
   → Cliente crea una orden → OrdenCreada
   → Cliente paga → PagoProcesado
   → Almacén envía → OrdenEnviada
  
   Las APIs y bases de datos son consecuencias, no el diseño principal.
```

### Principio #2: Granularidad Correcta

```
DEMASIADO FINO (event explosion):
  orden.item.agregado, orden.item.cantidad.cambiada,
  orden.direccion.calle.modificada, orden.cliente.nota.actualizada...
  → Cientos de tipos de eventos, imposible de mantener.

DEMASIADO GRUESO (event dios):
  datos.modificados → Un solo evento para todo.
  → Sin semántica, sin filtrado, acoplamiento masivo.

GRANULARIDAD CORRECTA:
  Eventos que representan una DECISIÓN DE NEGOCIO significativa.
  → orden.creada, orden.pagada, orden.enviada, orden.cancelada
  → Cada evento coincide con una transición de estado del aggregate.
```

### Principio #3: Nombrado de Eventos

```
CONVENCIÓN DE NOMBRADO:
  {agregado}.{verbo en pasado}
  
  ✅ orden.creada
  ✅ pago.procesado
  ✅ inventario.reservado
  ✅ envio.programado
  ✅ cliente.verificado
  
  ❌ OrdenCreada (PascalCase, inconsistente)
  ❌ ORDEN_CREADA (SCREAMING_CASE, parece constante)
  ❌ orden_creada (snake_case, inconsistente con el resto)
  ❌ crearOrden (imperativo, es un comando no un evento)
  ❌ datos_actualizados (genérico, sin significado)
  ❌ event.orden.creada (redundante)

TAXONOMÍA:
  {dominio}.{entidad}.{accion en pasado}
  
  ecommerce.orden.creada
  fintech.transaccion.completada
  iot.sensor.alerta_disparada
  usuarios.cuenta.verificada
```

### Principio #4: Cada Evento Tiene Dueño

```
Cada tipo de evento pertenece a UN SOLO bounded context / equipo.

Equipo Órdenes es dueño de:
  - orden.creada
  - orden.cancelada
  - orden.modificada

Equipo Pagos es dueño de:
  - pago.procesado
  - pago.rechazado

Solo el equipo dueño puede MODIFICAR el schema del evento.
Consumidores solicitan cambios via PR/review.
```

### Principio #5: Idempotencia desde el Diseño

```
NUNCA asumas "los mensajes se entregan exactamente una vez".
SIEMPRE diseña para "al menos una vez" con idempotencia.

Estrategias:
  1. eventId único + tabla de procesados
  2. UPSERT en vez de INSERT
  3. Operaciones conmutativas
  4. Tokens de idempotencia en comandos
  5. Versiones en aggregates (expectedVersion)
```

---

## 17.3 Decision Framework: Cuándo Usar EDA

```
¿Deberías usar Arquitectura Orientada a Eventos?

✅ USA EDA cuando:
  - Tienes múltiples servicios que necesitan reaccionar al mismo hecho
  - Necesitas alta resiliencia y desacoplamiento
  - Requieres auditoría y trazabilidad completa
  - Tu dominio es naturalmente asíncrono (e-commerce, IoT, fintech)
  - Escalas equipos independientes (cada equipo = un servicio)
  - Necesitas añadir nueva funcionalidad sin modificar servicios existentes

❌ NO USES EDA cuando:
  - Es una aplicación simple con un solo servicio
  - Necesitas consistencia fuerte inmediata (transacciones ACID)
  - El equipo es pequeño y no tiene experiencia en sistemas distribuidos
  - La complejidad adicional no se justifica por el beneficio
  - El volumen de eventos es muy bajo y la latencia crítica

⚠️ EDA PARCIAL (híbrido):
  - Usa EDA para comunicación entre servicios
  - Usa REST/gRPC para operaciones sincrónicas (consultas, autenticación)
  - Usa BD transaccional dentro de cada servicio
  - El mejor de los dos mundos: pragmatismo > purismo
```

---

## 17.4 Checklist de Madurez EDA

```
NIVEL 0 - Inconsciente:
  [ ] No hay eventos. Todo es REST/gRPC síncrono.

NIVEL 1 - Notificaciones:
  [ ] Publicamos algunos eventos "por si acaso"
  [ ] Los eventos no tienen schema formal
  [ ] No hay idempotencia
  [ ] No hay DLQ

NIVEL 2 - Eventos con Estado:
  [ ] Eventos con payload completo (auto-contenidos)
  [ ] Nombrado consistente (pasado, verbo, dominio)
  [ ] Schema Registry con compatibilidad backward
  [ ] Idempotencia implementada en consumidores
  [ ] DLQ configurada

NIVEL 3 - Streaming y Proyecciones:
  [ ] Event Sourcing en aggregates críticos
  [ ] Proyecciones materializadas (CQRS)
  [ ] Replay de eventos para nuevas proyecciones
  [ ] Snapshots para rendimiento
  [ ] Consumer groups independientes

NIVEL 4 - Madurez Total:
  [ ] Sagas para flujos complejos
  [ ] CloudEvents para interoperabilidad
  [ ] Trazas distribuidas (OpenTelemetry) en todos los servicios
  [ ] Consumer-driven contract tests
  [ ] Gobierno de schemas por equipo
  [ ] Monitoreo proactivo (lag, DLQ, latencia, throughput)
  [ ] Playbook de recuperación ante desastres
  [ ] Simulacros de chaos engineering
```

---

## 17.5 Principios Finales

```
1. EMPIEZA SIMPLE. Evoluciona.
   No necesitas Event Sourcing + CQRS + Saga + Schema Registry el día 1.
   Empieza con eventos simples de notificación. Agrega complejidad cuando la necesites.

2. LOS EVENTOS SON LA FUENTE DE VERDAD.
   Si hay conflicto entre la BD y los eventos, los eventos ganan.
   Son inmutables. Son historia. No se borran, no se modifican.

3. PUBLICA LO QUE PASÓ, NO LO QUE QUIERES QUE PASE.
   Evento = hecho pasado. Comando = intención futura.
   No los confundas. Es el error más común y más costoso.

4. CADA SERVICIO ES AUTÓNOMO.
   Tiene su propia BD, su propia lógica, su propio ciclo de deploy.
   No comparte BD con otros servicios. No llama a APIs de otros servicios.
   Solo escucha eventos.

5. DISEÑA PARA EL FALLO.
   Los mensajes se duplican. Los servicios caen. La red se parte.
   Idempotencia. DLQ. Reintentos. Circuit breakers. Compensación.
   No son opcionales. Son la base.

6. HAZ VISIBLE LO INVISIBLE.
   Trazas distribuidas. Métricas de lag. Alertas de DLQ.
   Si no puedes ver qué pasa en tu sistema de eventos, no tienes un sistema de eventos.
   Tienes un agujero negro.

7. LA ARQUITECTURA REFLEJA LA ORGANIZACIÓN.
   (Ley de Conway)
   Si tus equipos no son autónomos, tu EDA no será autónoma.
   Alinea bounded contexts con equipos. Un equipo = dueño de sus eventos.
```

---

## Resumen del Libro

Hemos recorrido el camino completo de la Arquitectura Orientada a Eventos:

1. **Fundamentos**: qué es EDA, eventos vs comandos, topologías, modelo de madurez.
2. **Tecnologías**: Kafka, RabbitMQ, NATS, AWS SQS/SNS/EventBridge. Cada una con su propósito.
3. **Patrones avanzados**: Domain Events + DDD, Event Sourcing, CQRS, Sagas, Consistencia Eventual.
4. **Implementación**: sistema e-commerce completo con compensación, idempotencia y proyecciones.
5. **Operaciones**: Schema Evolution, Testing, Observabilidad.
6. **Sabiduría**: Anti-patrones y principios.

EDA no es una tecnología. Es un modelo mental. Una forma de pensar en sistemas como flujos de eventos en lugar de estados estáticos. Como toda herramienta poderosa, requiere disciplina. Pero bien aplicada, produce sistemas más resilientes, escalables y mantenibles.

> "El viaje de mil eventos comienza con un solo publish." — Proverbio EDA

---

**Fin del libro.**

---

← [Capítulo anterior](16-observabilidad.md) | [Inicio](README.md)
