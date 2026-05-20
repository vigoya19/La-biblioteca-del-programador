# Capítulo 14: Schema Evolution y Compatibilidad

Los eventos son contratos. Cuando evolucionan —y siempre evolucionan—, debes garantizar que productores y consumidores puedan coexistir sin romperse. Schema Registry, formatos de serialización y estrategias de migración son el seguro de vida de tu sistema EDA.

> "El único sistema que no cambia es el que no se usa." — Ley de Lehman

---

## 14.1 El Problema del Versionado de Eventos

```
Versión 1 de OrdenCreada (en producción):
{
  type: "orden.creada",
  data: {
    ordenId: "123",
    clienteId: "456",
    total: 159.98
  }
}

Se añade un campo requerido: direccionEnvio
NUEVO Servicio de Envíos necesita direccionEnvio.

VERSIÓN 2:
{
  type: "orden.creada",
  version: 2,
  data: {
    ordenId: "123",
    clienteId: "456",
    total: 159.98,
    direccionEnvio: { calle: "...", ciudad: "..." }  // NUEVO
  }
}

PROBLEMAS:
1. Backward compatibility: ¿El consumidor antiguo puede leer v2?
2. Forward compatibility: ¿El consumidor nuevo puede leer v1?
3. Coexistencia: ¿Pueden v1 y v2 correr en producción simultáneamente?
4. Migración: ¿Cómo actualizamos todos los servicios sin downtime?
```

---

## 14.2 Schema Registry

El Schema Registry es la fuente de verdad de los esquemas de eventos. Centraliza la definición y validación de schemas.

```
┌──────────────────────────────────────────────────────────────┐
│                     SCHEMA REGISTRY                           │
│                                                              │
│  ┌─────────────────┐  ┌─────────────────┐                   │
│  │ OrdenCreada v1  │  │ OrdenCreada v2  │                   │
│  │  - ordenId      │  │  - ordenId      │                   │
│  │  - clienteId    │  │  - clienteId    │                   │
│  │  - total        │  │  - total        │                   │
│  │                 │  │  - direccionEnvio│ (nuevo, opcional)│
│  └─────────────────┘  └─────────────────┘                   │
│                                                              │
│  ┌─────────────────┐  ┌─────────────────┐                   │
│  │ PagoProcesado v1│  │InventarioRes v1 │                   │
│  └─────────────────┘  └─────────────────┘                   │
└──────────────────────────────────────────────────────────────┘
```

### Formatos de Serialización

| Formato | Ventajas | Desventajas | Cuándo usarlo |
|---------|----------|-------------|---------------|
| **Avro** | Schema embebido (ID), evolución robusta, compacto | Curva de aprendizaje, tooling | Kafka (estándar de facto) |
| **Protobuf** | Muy compacto, rápido, tipado fuerte | Schema evolution más rígido, necesita código generado | gRPC, alta performance |
| **JSON Schema** | Legible por humanos, sin tooling especial | Verboso, sin evolución nativa | Equipos que empiezan con EDA |

### Avro con Schema Registry (Kafka)

```json
// Schema registrado: OrdenCreada v1
{
  "type": "record",
  "name": "OrdenCreada",
  "namespace": "com.ecommerce.ordenes",
  "fields": [
    { "name": "ordenId", "type": "string" },
    { "name": "clienteId", "type": "string" },
    { "name": "total", "type": "double" },
    { "name": "items", "type": {
      "type": "array",
      "items": {
        "type": "record",
        "name": "Item",
        "fields": [
          { "name": "productoId", "type": "string" },
          { "name": "cantidad", "type": "int" },
          { "name": "precio", "type": "double" }
        ]
      }
    }}
  ]
}

// Schema evolucionado: OrdenCreada v2
// Añade direccionEnvio con valor por defecto para compatibilidad backward
{
  "type": "record",
  "name": "OrdenCreada",
  "namespace": "com.ecommerce.ordenes",
  "fields": [
    { "name": "ordenId", "type": "string" },
    { "name": "clienteId", "type": "string" },
    { "name": "total", "type": "double" },
    { "name": "items", "type": { "type": "array", "items": "Item" } },
    { "name": "direccionEnvio", "type": ["null", {
      "type": "record",
      "name": "Direccion",
      "fields": [
        { "name": "calle", "type": "string" },
        { "name": "ciudad", "type": "string" },
        { "name": "codigoPostal", "type": "string" }
      ]
    }], "default": null }  // ← Clave: default permite leer v1 como v2
  ]
}
```

```typescript
// Productor v2: envía eventos con Schema Registry
import { SchemaRegistry } from "@kafkajs/confluent-schema-registry";

const registry = new SchemaRegistry({ host: "http://schema-registry:8081" });

// Codificar evento con Avro (automáticamente registra/valida el schema)
const eventoAvro = await registry.encode(100, { // 100 = subject ID
  ordenId: "123",
  clienteId: "456",
  total: 159.98,
  items: [{ productoId: "P1", cantidad: 2, precio: 49.99 }],
  direccionEnvio: { calle: "Mayor 1", ciudad: "Madrid", codigoPostal: "28013" },
});

await producer.send({
  topic: "ordenes",
  messages: [{ value: eventoAvro }],
});

// Consumidor v1 (sin direccionEnvio): Avro aplica el default (null)
// Sin cambios en el código del consumidor v1
const eventoV1 = await registry.decode(await mensaje.value);
// eventoV1.direccionEnvio === null ← el default del schema
```

---

## 14.3 Tipos de Compatibilidad

```
BACKWARD COMPATIBILITY (hacia atrás)
─────────────────────────────────────
Consumidores con schema VIEJO pueden leer eventos con schema NUEVO.
Regla: Solo añadir campos con default, no eliminar campos requeridos.

✅ Añadir campo con default: { "name": "email", "type": "string", "default": "" }
✅ Añadir campo opcional (union con null): ["null", "string"]
❌ Eliminar campo requerido
❌ Cambiar tipo de campo (string → int)


FORWARD COMPATIBILITY (hacia adelante)
──────────────────────────────────────
Consumidores con schema NUEVO pueden leer eventos con schema VIEJO.
Regla: Campos nuevos deben tener default. Eliminar campos requiere versión nueva.

✅ Campos nuevos con default
✅ Campos opcionales
❌ Añadir campo sin default
❌ Cambiar tipo de campo


FULL COMPATIBILITY (completa)
─────────────────────────────
Backward + Forward. La más segura, la más restrictiva.

Solo permite:
✅ Añadir campos opcionales (con default)
✅ Eliminar campos opcionales
```

### Configuración en Schema Registry

```bash
# Configurar nivel de compatibilidad por subject
curl -X PUT http://schema-registry:8081/config/ordenes-value \
  -H "Content-Type: application/json" \
  -d '{"compatibility": "FULL"}'

# Verificar si un schema nuevo es compatible
curl -X POST http://schema-registry:8081/compatibility/subjects/ordenes-value/versions/latest \
  -H "Content-Type: application/json" \
  -d '{"schema": "{...}"}'
```

---

## 14.4 Estrategias de Migración

### Upcasting (Transform-on-Read)

```typescript
// Convertir eventos viejos al formato nuevo al leerlos
// El consumidor siempre trabaja con la versión más reciente

class EventUpcaster {
  upcast(evento: DomainEvent): DomainEvent {
    switch (evento.type) {
      case "orden.creada": {
        const data = evento.data as OrdenCreadaAnyVersion;

        // v1 → v2: añadir direccionEnvio si no existe
        if (!data.direccionEnvio) {
          return {
            ...evento,
            data: {
              ...data,
              direccionEnvio: data.direccionEnvio ?? {
                calle: "DESCONOCIDA",
                ciudad: "DESCONOCIDA",
                codigoPostal: "00000",
              },
            },
          };
        }

        // v2 ya es la actual
        return evento;
      }

      default:
        return evento;
    }
  }
}

// Uso en el consumidor
const upcaster = new EventUpcaster();
const eventoActualizado = upcaster.upcast(eventoOriginal);
// Ahora procesar con el handler que espera la última versión
```

### Versionado Dual (Coexistencia)

```typescript
// Productor publica ambas versiones durante la migración
async function publicarOrdenCreada(orden: Orden): Promise<void> {
  const eventV1: OrdenCreadaV1 = {
    type: "orden.creada",
    data: { ordenId: orden.id, clienteId: orden.clienteId, total: orden.total },
  };

  const eventV2: OrdenCreadaV2 = {
    type: "orden.creada.v2",
    data: { ...eventV1.data, direccionEnvio: orden.direccionEnvio },
  };

  // Publicar ambas versiones en topics separados
  await eventBus.publicar("ordenes.v1", eventV1);
  await eventBus.publicar("ordenes.v2", eventV2);
}

// Consumidores migran gradualmente de v1 a v2
// Semana 1-2: consumidores escuchan solo v1
// Semana 3: nuevos consumidores escuchan v2. Viejos siguen en v1.
// Semana 4: todos en v2. Se depreca publicación de v1.
// Semana 5: eliminar v1.
```

### Transform-on-Read con BD de esquemas

```typescript
// Versión más robusta: almacenar mapas de transformación

const UPCASTERS: Record<string, (data: unknown) => unknown> = {
  "orden.creada": (data: any) => {
    let migrado = { ...data };

    // Migración v1 → v2
    if (!migrado.direccionEnvio) {
      migrado.direccionEnvio = {
        calle: data.direccion?.calle ?? "N/A",
        ciudad: data.direccion?.ciudad ?? "N/A",
        codigoPostal: data.direccion?.cp ?? "00000",
      };
    }

    // Migración v2 → v3 (futura)
    // if (!migrado.tipoEnvio) { migrado.tipoEnvio = "estandar"; }

    return migrado;
  },
};
```

---

## 14.5 CloudEvents: Estándar de Eventos

CloudEvents es una especificación CNCF que define un formato común para eventos, facilitando la interoperabilidad entre sistemas:

```typescript
// Ejemplo de evento CloudEvents
const cloudEvent = {
  // ─── Atributos requeridos ───
  "specversion": "1.0",           // Versión de la especificación
  "type": "com.ecommerce.orden.creada",  // Tipo de evento (reverse-DNS)
  "source": "/servicios/ordenes",       // Origen del evento
  "id": "a1b2c3d4-e5f6-7890",           // ID único del evento

  // ─── Atributos opcionales ───
  "time": "2024-01-15T10:30:00Z",
  "datacontenttype": "application/json",
  "subject": "orden-123",
  "dataschema": "https://schema-registry.ecommerce.com/orden-creada/v2",

  // ─── Extensiones ───
  "traceparent": "00-traceId-spanId-01",
  "partitionkey": "orden-123",

  // ─── Payload ───
  "data": {
    "ordenId": "123",
    "clienteId": "456",
    "total": 159.98,
    "items": [
      { "productoId": "P1", "cantidad": 2, "precio": 49.99 }
    ]
  }
};
```

```typescript
// Implementar CloudEvents en tu sistema
import { CloudEvent, HTTP } from "cloudevents";

// Productor
const ce = new CloudEvent({
  type: "com.ecommerce.orden.creada",
  source: "/servicios/ordenes",
  data: { ordenId: "123", clienteId: "456", total: 159.98 },
});

// Enviar via HTTP (modo binario: headers + body)
const message = HTTP.binary(ce);
// Headers: ce-type, ce-source, ce-id, content-type
// Body: JSON del data

// Consumidor
const receivedEvent = HTTP.toEvent({
  headers: req.headers,
  body: req.body,
});
// receivedEvent.data → { ordenId: "123", ... }
```

---

## 14.6 Gobierno de Esquemas en Equipos Grandes

```
PRINCIPIOS DE GOBIERNO
───────────────────────
1. Cada evento tiene un DUEÑO (owner team)
   - Solo el equipo dueño puede cambiar el schema
   - Los consumidores solicitan cambios vía PR/review

2. Schema Registry como Single Source of Truth
   - Schemas viven en git (IaC) → deployados a Schema Registry
   - CI/CD valida compatibilidad antes del deploy

3. Convención de nombrado
   - Subject: {dominio}.{entidad}.{version}
   - Event type: reverse-DNS
   - Ej: com.ecommerce.ordenes.OrdenCreada

4. Estrategia de compatibilidad por subject
   - Eventos core (muchos consumidores): FULL
   - Eventos internos (un consumidor): BACKWARD
   - Eventos efímeros (log, métricas): NONE

5. Proceso de cambio de schema
   ┌─────────────────────────────────┐
   │ 1. PR con schema + changelog    │
   │ 2. CI valida compatibilidad     │
   │ 3. Review del equipo dueño      │
   │ 4. Deploy schema al registry    │
   │ 5. Deploy productores           │
   │ 6. Migrar consumidores          │
   │ 7. Deprecar versión antigua     │
   └─────────────────────────────────┘
```

---

## Resumen del Capítulo

- Los **schemas de eventos** son contratos que evolucionan. Planifica la evolución desde el día 1.
- El **Schema Registry** centraliza la definición, validación y compatibilidad de schemas.
- **Avro** es el estándar de facto para Kafka: compacto, con schema evolution robusto y valores por defecto.
- **Tipos de compatibilidad**: Backward (viejo lee nuevo), Forward (nuevo lee viejo), Full (ambos). Full es la más segura.
- **Estrategias de migración**: upcasting (transformar al leer), versionado dual (publicar ambas versiones), transform-on-read.
- **CloudEvents** estandariza el envelope de eventos para interoperabilidad entre sistemas.
- El **gobierno de schemas** define dueños, procesos de cambio y validación automatizada en CI/CD.

En el siguiente capítulo exploramos estrategias de testing para sistemas event-driven.
