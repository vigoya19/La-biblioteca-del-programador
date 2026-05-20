# Capítulo 9: Patrones de Comunicación

> "En un sistema distribuido, la comunicación lo es todo. Hazla bien o todo se desmorona."

## 9.1 Comunicación Síncrona vs Asíncrona

| | Síncrona | Asíncrona |
|---|----------|----------|
| **El cliente** | Espera respuesta | No espera |
| **Acoplamiento** | Temporal (ambos deben estar vivos) | Solo espacial |
| **Resiliencia** | Fallo en cascada si destino falla | Degradación elegante |
| **Complejidad** | Simple de razonar | Difícil de debuggear |
| **Ejemplos** | REST, gRPC, GraphQL | Kafka, RabbitMQ, SQS |

> **Regla empírica**: Si la operación es una consulta, usa síncrono. Si es un comando que dispara un proceso, considera asíncrono.

## 9.2 REST (Representational State Transfer)

No es solo "HTTP con JSON". Es un estilo arquitectónico con restricciones:

### Restricciones de REST
1. **Cliente-Servidor**: Separación de responsabilidades.
2. **Stateless**: Cada request contiene toda la información necesaria.
3. **Cacheable**: Las respuestas deben declarar si son cacheadas.
4. **Interfaz Uniforme**: Recursos identificados por URIs, manipulación vía representaciones.
5. **Sistema en Capas**: El cliente no sabe si habla con el servidor final o un proxy.
6. **Code on Demand** (opcional): Servidor puede enviar código ejecutable.

### Diseño de Recursos

```
# Colección
GET    /api/pedidos          → Lista pedidos
POST   /api/pedidos          → Crea pedido

# Recurso individual
GET    /api/pedidos/123      → Obtiene pedido
PUT    /api/pedidos/123      → Reemplaza pedido
PATCH  /api/pedidos/123      → Actualiza parcialmente
DELETE /api/pedidos/123      → Elimina pedido

# Sub-recursos
GET    /api/pedidos/123/items          → Items del pedido
GET    /api/pedidos/123/items/5        → Item específico
```

### Niveles de Madurez REST (Richardson Maturity Model)

| Nivel | Descripción |
|-------|-------------|
| 0 | Una URL, un método (RPC sobre HTTP) |
| 1 | Múltiples URLs por recurso |
| 2 | Verbos HTTP correctos + códigos de estado |
| 3 | HATEOAS (hypermedia controls) |

La mayoría de APIs "REST" son realmente nivel 2. El nivel 3 es poco común.

### Códigos de Estado Esenciales

```
2xx: Éxito
  200 OK, 201 Created, 204 No Content

3xx: Redirección
  301 Moved Permanently, 304 Not Modified

4xx: Error del Cliente
  400 Bad Request, 401 Unauthorized, 403 Forbidden
  404 Not Found, 409 Conflict, 422 Unprocessable Entity
  429 Too Many Requests

5xx: Error del Servidor
  500 Internal Server Error, 502 Bad Gateway
  503 Service Unavailable, 504 Gateway Timeout
```

## 9.3 GraphQL

Lenguaje de consulta para APIs que permite al cliente pedir exactamente lo que necesita.

```graphql
# El cliente especifica exactamente qué datos quiere
query {
  pedido(id: "123") {
    id
    total
    estado
    cliente {
      nombre
      email
    }
    items {
      producto { nombre }
      cantidad
      precio
    }
  }
}
```

### Ventajas sobre REST
- **No over-fetching**: Cliente pide solo lo que necesita.
- **No under-fetching**: Una sola llamada obtiene datos anidados.
- **Tipado fuerte**: Schema explícito.
- **Evolución sin versionado**: Añadir campos no rompe clientes existentes.

### Desventajas
- **Complejidad del servidor**: Resolver queries anidadas es complejo.
- **Caching difícil**: Todo es POST, no se beneficia de HTTP caching.
- **Consultas costosas**: Un cliente puede pedir datos que generan N+1 queries.
- **Rate limiting complejo**: No es trivial limitar por complejidad de query.

### Cuándo Usar GraphQL
- Aplicaciones móviles/web con necesidades de datos variables.
- Múltiples clientes con distintas necesidades de datos.
- Datos altamente interconectados (grafos).

## 9.4 API Composition Pattern

En microservicios, los datos están dispersos. El API Composer orquesta llamadas a múltiples servicios y ensambla la respuesta.

```
Cliente ──► API Gateway ──► Servicio Usuarios
                │
                ├──────────► Servicio Pedidos
                │
                └──────────► Servicio Pagos
                     (ensambla respuesta)
```

## 9.5 Backend for Frontend (BFF)

Una API específica para cada tipo de cliente.

```
App Móvil  ──► BFF Móvil  ──► Microservicios
Web App    ──► BFF Web    ──► Microservicios
IoT Device ──► BFF IoT    ──► Microservicios
```

Cada BFF optimiza: datos agregados, formato, protocolo.

## 9.6 Service Discovery

En entornos dinámicos (Kubernetes, cloud), las IPs cambian. Necesitas descubrir servicios.

### Client-Side Discovery
```
Servicio A ──► Service Registry ──► Lista de instancias de B
     │                                    │
     └─────── balancea y llama ───────────┘
```
Ej: Eureka, Consul.

### Server-Side Discovery
```
Servicio A ──► Load Balancer ──► Service Registry
                     │
                     └─────► Servicio B (instancia)
```
Ej: AWS ALB, Kubernetes Service.

## 9.7 Service Mesh

Capa de infraestructura que maneja la comunicación entre servicios de forma transparente.

```
Servicio A ──► Sidecar Proxy (Envoy) ──► Sidecar Proxy (Envoy) ──► Servicio B
                     │                            │
                     └────────── Istio ───────────┘
                     (mTLS, retry, circuit breaking,
                      métricas, tracing, routing)
```

**Cuándo usar Service Mesh**: +50 microservicios, necesidades avanzadas de tráfico, seguridad zero-trust.

---

> **Reflexión del capítulo**: La comunicación es el sistema nervioso de tu arquitectura. Cada milisegundo de latencia, cada fallo de red, cada timeout afecta la experiencia del usuario. Diseña la comunicación con la misma seriedad que el dominio.

---

← [Capítulo anterior](08-estilos-arquitectonicos.md) | [Inicio](README.md) | [Capítulo siguiente →](10-mensajeria-eventos.md)
