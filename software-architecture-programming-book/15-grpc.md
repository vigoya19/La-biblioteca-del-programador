# Capítulo 15: gRPC y Protocol Buffers

> "Cuando el rendimiento importa, REST no es suficiente."

## 15.1 ¿Qué es gRPC?

gRPC (gRPC Remote Procedure Call) es un framework de RPC de alto rendimiento creado por Google. Usa **HTTP/2** como transporte y **Protocol Buffers** como formato de serialización y lenguaje de definición de interfaz.

**Llamas a un método remoto como si fuera local.**

```protobuf
// Definición del servicio (archivo .proto)
service OrderService {
  rpc CreateOrder (CreateOrderRequest) returns (CreateOrderResponse);
  rpc GetOrder (GetOrderRequest) returns (Order);
  rpc ListOrders (ListOrdersRequest) returns (stream Order);  // Server streaming
}
```

## 15.2 Protocol Buffers (Protobuf)

Formato de serialización binario, fuertemente tipado y con esquema explícito.

```protobuf
syntax = "proto3";

message Order {
  string id = 1;
  string customer_id = 2;
  repeated OrderItem items = 3;
  double total = 4;
  Status status = 5;

  enum Status {
    UNKNOWN = 0;
    PENDING = 1;
    CONFIRMED = 2;
    SHIPPED = 3;
  }
}

message CreateOrderRequest {
  string customer_id = 1;
  repeated OrderItem items = 2;
}
```

### Ventajas sobre JSON
| | Protobuf | JSON |
|---|----------|------|
| **Tamaño** | 3-10x más pequeño | Verboso |
| **Velocidad** | 2-7x más rápido en serializar | Más lento |
| **Tipado** | Estricto (schema) | Débil |
| **Evolución** | Hacia atrás/adelante nativa | Manual |
| **Legibilidad** | No legible por humanos | Legible |
| **Herramientas** | Generación de código automática | Manual |

## 15.3 Tipos de Streaming en gRPC

```
1. Unary (clásico):
   Cliente ──req──► Servidor
          ◄──res──

2. Server Streaming:
   Cliente ──req──► Servidor
          ◄──res1─
          ◄──res2─
          ◄──res3─

3. Client Streaming:
   Cliente ──req1─►
          ──req2─► Servidor
          ──req3─►
          ◄──res──

4. Bidirectional Streaming:
   Cliente ──req1─►
          ◄──res1─
          ──req2─► Servidor
          ◄──res2─
```

### Casos de Uso para Cada Tipo

| Tipo | Casos de Uso |
|------|-------------|
| **Unary** | APIs CRUD, autenticación |
| **Server Streaming** | Descarga de reportes grandes, feeds de datos |
| **Client Streaming** | Upload de archivos grandes, ingestión de logs |
| **Bidirectional** | Chat en tiempo real, gaming, colaboración |

## 15.4 gRPC vs REST: ¿Cuándo Elegir Qué?

| | gRPC | REST |
|---|------|------|
| **Rendimiento** | Excelente (binario + HTTP/2) | Bueno (JSON + HTTP/1.1 o 2) |
| **Browser support** | Limitado (necesita grpc-web) | Nativo |
| **Debugging** | Necesitas herramientas (grpcurl) | curl, navegador, Postman |
| **Ecosistema** | Creciente | Maduro, universal |
| **Schema** | Protobuf (.proto) | OpenAPI/Swagger |
| **Streaming** | Nativo y robusto | SSE/WebSocket (añadidos) |
| **Load Balancing** | Requiere HTTP/2-aware LB | Universal |
| **Caching** | No usa HTTP caching | Headers HTTP estándar |

### Usa gRPC para:
- Comunicación **service-to-service** de alta frecuencia.
- Microservicios con necesidades de baja latencia.
- Streaming bidireccional.
- Ambientes polyglot (generación automática de clientes en 11+ lenguajes).

### Usa REST para:
- APIs públicas o de terceros.
- Clientes web/móviles tradicionales.
- Cuando el ecosistema de herramientas REST es necesario.

## 15.5 gRPC Interceptors

Middleware en gRPC. Equivalente a filtros HTTP:

```java
// Interceptor de autenticación
public class AuthInterceptor implements ServerInterceptor {
    @Override
    public <ReqT, RespT> Listener<ReqT> interceptCall(
            ServerCall<ReqT, RespT> call,
            Metadata headers,
            ServerCallHandler<ReqT, RespT> next) {

        String token = headers.get(Metadata.Key.of("authorization", ...));
        if (!isValid(token)) {
            call.close(Status.UNAUTHENTICATED, new Metadata());
            return new Listener<>() {};
        }
        return next.startCall(call, headers);
    }
}
```

## 15.6 gRPC en Producción

### Health Checking
gRPC tiene un protocolo estándar de health check que Kubernetes entiende nativamente.

### Load Balancing
- **Proxy LB**: Envoy, Linkerd, Nginx con soporte gRPC.
- **Client-side LB**: gRPC tiene load balancing del lado del cliente con service discovery.

### Deadlines / Timeouts
Siempre configura deadlines en gRPC. Sin deadline, un request puede colgarse para siempre.

```java
stub.withDeadline(Deadline.after(5, TimeUnit.SECONDS)).createOrder(request);
```

---

> **Reflexión del capítulo**: gRPC no es un reemplazo universal de REST, es una herramienta para escenarios específicos. La regla práctica: REST para APIs externas, gRPC para comunicación interna de alto rendimiento. Conoce ambos y usa el correcto en cada contexto.

---

← [Capítulo anterior](14-diseno-apis.md) | [Inicio](README.md) | [Capítulo siguiente →](16-seguridad.md)
