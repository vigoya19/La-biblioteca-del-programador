# Capítulo 14: Diseño de APIs

> "Tu API es la interfaz de usuario del desarrollador. Diseñala con empatía."

## 14.1 Principios del Buen Diseño de APIs

### 1. API-First Design
Diseña el contrato antes de implementar. Usa especificaciones como OpenAPI, AsyncAPI, GraphQL Schema o Protobuf.

**Beneficio**: Frontend y backend trabajan en paralelo contra el contrato.

### 2. Naming Consistente
```
✅ BIEN:
GET  /api/v1/users
GET  /api/v1/users/{id}
POST /api/v1/users

❌ MAL:
GET  /api/v1/getUsers
POST /api/v1/create_user
GET  /api/v1/USER_INFO?id=5
```

### 3. Versionado desde el Día 1
```
# En URL (más común, fácil de cachear)
/api/v1/users
/api/v2/users

# En header (más REST puro)
Accept: application/vnd.empresa.v2+json

# Query param (menos recomendado)
/api/users?version=2
```

### 4. Paginación Estandarizada
```json
// Request
GET /api/v1/orders?page=2&size=20&sort=createdAt,desc

// Response
{
  "data": [...],
  "pagination": {
    "page": 2,
    "size": 20,
    "totalElements": 1547,
    "totalPages": 78
  },
  "_links": {
    "first": "/api/v1/orders?page=0&size=20",
    "prev": "/api/v1/orders?page=1&size=20",
    "next": "/api/v1/orders?page=3&size=20",
    "last": "/api/v1/orders?page=77&size=20"
  }
}
```

**Alternativa: Cursor-based pagination** para datos en tiempo real (feeds, timelines):
```json
GET /api/v1/orders?cursor=eyJpZCI6MTIzfQ&limit=20
```

## 14.2 Formato de Errores Consistente

```json
{
  "error": {
    "code": "ORDER_NOT_FOUND",
    "message": "El pedido con ID 12345 no existe",
    "details": [
      {
        "field": "orderId",
        "reason": "El ID no corresponde a ningún pedido activo"
      }
    ],
    "traceId": "abc-123-def",
    "timestamp": "2024-01-15T10:30:00Z"
  }
}
```

¿Por qué `traceId`? Porque en producción, el usuario te va a pasar el mensaje de error, y con el traceId puedes buscar en los logs.

## 14.3 Filtrado, Búsqueda y Ordenamiento

```
# Filtros simples
GET /api/v1/products?status=ACTIVE&category=electronics

# Filtros avanzados
GET /api/v1/products?filter=price>100 AND price<500 AND (category=electronics OR category=computers)

# Búsqueda textual
GET /api/v1/products?search=macbook pro

# Ordenamiento
GET /api/v1/products?sort=-price,+name  # -desc, +asc
```

## 14.4 Idempotencia y Seguridad

### Idempotency Keys
Para operaciones que no deben duplicarse (pagos, creación de órdenes):

```
POST /api/v1/payments
Idempotency-Key: 550e8400-e29b-41d4-a716-446655440000

Si el request se envía 3 veces (timeout, retry), el servidor:
1. Reconoce la key → devuelve resultado del primer intento.
2. No crea pagos duplicados.
```

### Rate Limiting Headers
```
HTTP/1.1 200 OK
X-RateLimit-Limit: 1000
X-RateLimit-Remaining: 743
X-RateLimit-Reset: 1705314000
Retry-After: 60
```

### Seguridad en APIs
- Siempre HTTPS (nunca HTTP en producción).
- Autenticación con Bearer tokens (JWT, OAuth2).
- Rate limiting por API key/usuario.
- Validación estricta de inputs (nunca confiar en el cliente).
- CORS configurado explícitamente (no `*` en producción).

## 14.5 Documentación de APIs

### OpenAPI (Swagger)
```yaml
openapi: 3.0.0
info:
  title: API de Pedidos
  version: 1.0.0
paths:
  /orders:
    post:
      summary: Crear un pedido
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/CreateOrderRequest'
      responses:
        '201':
          description: Pedido creado exitosamente
        '400':
          $ref: '#/components/responses/BadRequest'
        '401':
          $ref: '#/components/responses/Unauthorized'
```

**Herramientas**: Swagger UI, Redoc, Stoplight, Postman Collections.

## 14.6 APIs Asíncronas (AsyncAPI)

Cuando tu API no es REST sino eventos:

```yaml
asyncapi: 2.6.0
info:
  title: Eventos de Pedidos
channels:
  orders/created:
    publish:
      message:
        payload:
          type: object
          properties:
            orderId: string
            customerId: string
            total: number
```

## 14.7 Patrones Avanzados de API

### Operaciones Bulk
Cuando el cliente necesita crear/actualizar múltiples recursos:

```json
// ❌ MAL: 100 llamadas individuales
for (item in items) { POST /api/v1/products item }

// ✅ BIEN: Bulk endpoint
POST /api/v1/products/bulk
{
  "operations": [
    { "action": "create", "data": { "name": "Producto 1", ... } },
    { "action": "update", "id": "456", "data": { "price": 99.99 } },
    { "action": "delete", "id": "789" }
  ]
}

// Respuesta: resultado individual por cada operación
{
  "results": [
    { "status": 201, "id": "new-123" },
    { "status": 200, "id": "456" },
    { "status": 404, "error": "Product not found" }
  ],
  "summary": { "total": 3, "succeeded": 2, "failed": 1 }
}
```

### Long-Running Operations
Para operaciones que toman más de 30 segundos:

```
POST /api/v1/reports/generate
→ 202 Accepted
  Location: /api/v1/operations/abc-123

GET /api/v1/operations/abc-123
→ 200 OK
  { "status": "PROCESSING", "progress": 65 }

GET /api/v1/operations/abc-123 (cuando termina)
→ 303 See Other
  Location: /api/v1/reports/jan-2024.pdf
```

### Sparse Fields
Permite al cliente seleccionar qué campos quiere:

```
GET /api/v1/products/123?fields=id,name,price,images(url)
{
  "id": "123",
  "name": "Laptop Pro",
  "price": 999.99,
  "images": [
    { "url": "https://cdn.example.com/img1.jpg" }
  ]
}
```

### Expansión de Relaciones (Side-loading)
```
GET /api/v1/orders/456?expand=customer,items.product
{
  "id": "456",
  "customer": { "id": "c1", "name": "Ana" },
  "items": [
    { "product": { "id": "p1", "name": "Laptop" }, "quantity": 1 }
  ]
}
```

## 14.8 Webhooks: Cuando Tú Llamas a Ellos

Para notificar eventos a sistemas externos:

```
1. El cliente registra un webhook:
   POST /api/v1/webhooks
   {
     "url": "https://cliente.com/hooks",
     "events": ["order.created", "order.shipped"],
     "secret": "whsec_xxx"  // Para firmar payloads
   }

2. Cuando el evento ocurre, tu sistema hace POST:
   POST https://cliente.com/hooks
   Headers:
     X-Webhook-Signature: sha256=abc123...
   Body:
   {
     "event": "order.created",
     "timestamp": "2024-01-15T10:30:00Z",
     "data": { "orderId": "123", ... }
   }

3. El cliente verifica la firma con el secret compartido.
```

**Buenas prácticas para Webhooks**:
- Reintentos con backoff exponencial (hasta 24h).
- Firma HMAC para verificar autenticidad.
- Idempotency key en cada entrega.
- Dashboard de webhooks para el cliente (estado, reintentos, logs).

## 14.9 Anti-Patrones de API

### 1. API Parlante (Chatty API)
```
❌ El cliente necesita 5 llamadas para renderizar una pantalla.
✅ API Composition: un endpoint que devuelve todo lo necesario.
```

### 2. API Omnisciente
```
❌ GET /api/v1/getAllUsersWithOrdersAndPaymentsAnd...
✅ Múltiples endpoints enfocados. O GraphQL si realmente se necesita.
```

### 3. Fuga de Abstracción de Base de Datos
```
❌ GET /api/v1/users?select=id,name&where=age>18&join=orders&group_by=city
✅ Endpoints semánticos que ocultan la estructura interna:
   GET /api/v1/users/search?min_age=18&include_recent_orders=true
```

### 4. Códigos de Estado Incorrectos
```
❌ return 200 OK { "error": "No autorizado" }
❌ return 500 { "message": "El producto no existe" }  // Debe ser 404
✅ Códigos HTTP semánticamente correctos siempre.
```

### 5. Autenticación en Query String
```
❌ GET /api/v1/orders?api_key=sk_live_abc123  (API key en URL → logs, caches, proxies)
✅ Header: Authorization: Bearer sk_live_abc123
```

### 6. N+1 Queries por Diseño
```
❌ GET /api/v1/orders  → devuelve IDs de cliente
   GET /api/v1/users/1 → el cliente hace N llamadas más
✅ GET /api/v1/orders?include=customer  → una sola llamada
```

### 7. Over-fetching Crónico
```
❌ GET /api/v1/users/me  → 4MB de datos que el cliente no pidió
✅ GET /api/v1/users/me?fields=name,email,avatar
✅ O implementa GraphQL si el problema es sistémico.
```

### 8. Timeouts Sin Respuesta
```
❌ El servidor procesa 45 segundos y el cliente recibe timeout.
✅ Patrón de operación de larga duración (202 + Location header).
```

## 14.10 Checklist de Calidad de API

Antes de publicar una API:

- [ ] ¿Todas las URLs usan sustantivos en plural, no verbos?
- [ ] ¿Los códigos de estado HTTP son semánticamente correctos?
- [ ] ¿Los errores tienen formato consistente con traceId?
- [ ] ¿Todos los endpoints tienen rate limiting?
- [ ] ¿El versionado está definido (v1 en URL o header)?
- [ ] ¿Las respuestas de colección incluyen paginación?
- [ ] ¿Los campos de fecha usan ISO 8601 (con timezone)?
- [ ] ¿Hay un health check endpoint (`GET /health`)?
- [ ] ¿La documentación OpenAPI está actualizada y publicada?
- [ ] ¿Tienen CORS configurado explícitamente?
- [ ] ¿Las operaciones de escritura aceptan Idempotency-Key?
- [ ] ¿Los IDs son UUIDs (no secuenciales expuestos)?
- [ ] ¿El límite de payload size está definido y documentado?
- [ ] ¿Existe un endpoint de deprecación para versiones viejas?

## 14.11 Estrategia de Deprecación

1. **Anunciar** con suficiente anticipación (mínimo 3-6 meses).
2. **Headers de aviso**: `Sunset: Sat, 31 Dec 2024 23:59:59 GMT`, `Deprecation: true`.
3. **Monitorizar uso**: Saber quién sigue usando la versión antigua.
4. **Comunicar proactivamente**: No es solo un header HTTP; envía emails a los consumidores.
5. **Mantener compatibilidad** hacia atrás mientras sea razonable.
6. **Sunset gradual**: Primero warnings en respuestas, luego rate limit más restrictivo, luego corte definitivo.
7. **Dar alternativas claras**: Cada endpoint deprecado debe tener documentado su reemplazo.

---

> **Reflexión del capítulo**: Diseñar APIs es diseño de producto para desarrolladores. Una API fea, inconsistente o mal documentada es una experiencia de usuario terrible. Trata a tus APIs con el mismo cuidado que tratarías una interfaz gráfica. Y recuerda: es más fácil diseñar bien desde el principio que arreglar una API con miles de consumidores.

---

← [Capítulo anterior](13-protocolos.md) | [Inicio](README.md) | [Capítulo siguiente →](15-grpc.md)
