# Capítulo 18: gRPC y Protocol Buffers en Go

> "gRPC es a los microservicios lo que HTTP/REST fue a la web: el estándar de comunicación de alto rendimiento."

## 18.1 ¿Por Qué gRPC en Go?

Go y gRPC comparten ADN: ambos nacieron en Google para resolver problemas de escala. Go es el lenguaje #1 para gRPC en producción (Kubernetes, etcd, Vitess, CockroachDB lo usan). La combinación ofrece:

- **Tipado fuerte**: Protobuf genera structs Go con validación en compilación.
- **Alto rendimiento**: HTTP/2 nativo, serialización binaria, streaming bidireccional.
- **Generación de código**: Un archivo `.proto` genera cliente y servidor en segundos.
- **Streaming nativo**: Server, client y bidirectional streaming sin WebSockets.

## 18.2 Setup — De Cero a Servidor gRPC

```bash
# 1. Instalar protoc (Protocol Buffers compiler)
# macOS:
brew install protobuf
# Linux:
apt install protobuf-compiler

# 2. Instalar plugins de Go
go install google.golang.org/protobuf/cmd/protoc-gen-go@latest
go install google.golang.org/grpc/cmd/protoc-gen-go-grpc@latest

# 3. Asegurar que $GOPATH/bin está en PATH
export PATH="$PATH:$(go env GOPATH)/bin"

# 4. Crear módulo Go
mkdir mi-servicio && cd mi-servicio
go mod init github.com/usuario/mi-servicio
```

### El Archivo .proto — La Fuente de Verdad

```protobuf
// api/v1/orders.proto
syntax = "proto3";

package orders.v1;

option go_package = "github.com/usuario/mi-servicio/api/v1/orderspb";

// ─── Servicio ───
service OrderService {
  rpc CreateOrder(CreateOrderRequest) returns (CreateOrderResponse);
  rpc GetOrder(GetOrderRequest) returns (Order);
  rpc ListOrders(ListOrdersRequest) returns (stream Order); // Server streaming
  rpc StreamOrders(stream OrderUpdate) returns (stream OrderEvent); // Bidireccional
}

// ─── Mensajes ───
message CreateOrderRequest {
  string customer_id = 1;
  repeated OrderItem items = 2;
}

message CreateOrderResponse {
  string order_id = 1;
  string status = 2;
}

message GetOrderRequest {
  string order_id = 1;
}

message Order {
  string order_id = 1;
  string customer_id = 2;
  repeated OrderItem items = 3;
  double total = 4;
  OrderStatus status = 5;
  int64 created_at = 6; // Unix timestamp
}

message OrderItem {
  string product_id = 1;
  string name = 2;
  int32 quantity = 3;
  double unit_price = 4;
}

message ListOrdersRequest {
  string customer_id = 1;
  int32 page_size = 2;
  string page_token = 3;
}

message OrderUpdate {
  string order_id = 1;
  OrderStatus new_status = 2;
}

message OrderEvent {
  string order_id = 1;
  OrderStatus status = 2;
  int64 timestamp = 3;
}

enum OrderStatus {
  ORDER_STATUS_UNSPECIFIED = 0; // Siempre tener un 0 para defaults
  ORDER_STATUS_PENDING = 1;
  ORDER_STATUS_CONFIRMED = 2;
  ORDER_STATUS_SHIPPED = 3;
  ORDER_STATUS_DELIVERED = 4;
  ORDER_STATUS_CANCELLED = 5;
}
```

### Generar Código Go

```bash
# Generar stubs desde el .proto
protoc \
  --go_out=. --go_opt=paths=source_relative \
  --go-grpc_out=. --go-grpc_opt=paths=source_relative \
  api/v1/orders.proto

# Esto genera:
# api/v1/orderspb/orders.pb.go       (structs de mensajes)
# api/v1/orderspb/orders_grpc.pb.go  (interfaces de servidor + cliente)
```

### Implementar el Servidor

```go
package main

import (
    "context"
    "log"
    "net"

    "google.golang.org/grpc"
    "google.golang.org/grpc/codes"
    "google.golang.org/grpc/status"

    pb "github.com/usuario/mi-servicio/api/v1/orderspb"
)

// server implementa OrderServiceServer
type server struct {
    pb.UnimplementedOrderServiceServer // Embebido para forward compat
    orders map[string]*pb.Order        // En producción: base de datos real
}

func (s *server) CreateOrder(ctx context.Context, req *pb.CreateOrderRequest) (*pb.CreateOrderResponse, error) {
    // Validación de negocio
    if req.CustomerId == "" {
        return nil, status.Error(codes.InvalidArgument, "customer_id requerido")
    }
    if len(req.Items) == 0 {
        return nil, status.Error(codes.InvalidArgument, "al menos un item requerido")
    }

    // Crear pedido
    orderID := generarID()
    order := &pb.Order{
        OrderId:    orderID,
        CustomerId: req.CustomerId,
        Items:      req.Items,
        Total:      calcularTotal(req.Items),
        Status:     pb.OrderStatus_ORDER_STATUS_PENDING,
        CreatedAt:  time.Now().Unix(),
    }
    s.orders[orderID] = order

    return &pb.CreateOrderResponse{
        OrderId: orderID,
        Status:  "PENDING",
    }, nil
}

func main() {
    lis, err := net.Listen("tcp", ":50051")
    if err != nil {
        log.Fatalf("failed to listen: %v", err)
    }

    s := grpc.NewServer()
    pb.RegisterOrderServiceServer(s, &server{
        orders: make(map[string]*pb.Order),
    })

    log.Println("gRPC server listening on :50051")
    if err := s.Serve(lis); err != nil {
        log.Fatalf("failed to serve: %v", err)
    }
}
```

### El Cliente

```go
package main

import (
    "context"
    "log"
    "time"

    "google.golang.org/grpc"
    "google.golang.org/grpc/credentials/insecure"

    pb "github.com/usuario/mi-servicio/api/v1/orderspb"
)

func main() {
    // Conectar al servidor
    conn, err := grpc.Dial("localhost:50051",
        grpc.WithTransportCredentials(insecure.NewCredentials()), // Solo para dev
        grpc.WithBlock(), // Esperar a que la conexión esté lista
    )
    if err != nil {
        log.Fatalf("no se pudo conectar: %v", err)
    }
    defer conn.Close()

    client := pb.NewOrderServiceClient(conn)

    // Llamada unaria
    ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
    defer cancel()

    resp, err := client.CreateOrder(ctx, &pb.CreateOrderRequest{
        CustomerId: "customer-123",
        Items: []*pb.OrderItem{
            {ProductId: "prod-1", Name: "Laptop", Quantity: 1, UnitPrice: 999.99},
        },
    })
    if err != nil {
        log.Fatalf("CreateOrder failed: %v", err)
    }
    log.Printf("Pedido creado: %s (%s)", resp.OrderId, resp.Status)
}
```

---

## 18.3 Los 4 Tipos de Streaming

### 1. Server Streaming — Servidor Envía Múltiples Respuestas

```protobuf
rpc ListOrders(ListOrdersRequest) returns (stream Order);
```

```go
func (s *server) ListOrders(req *pb.ListOrdersRequest, stream pb.OrderService_ListOrdersServer) error {
    for _, order := range s.orders {
        if order.CustomerId == req.CustomerId {
            // Enviar cada orden como un mensaje separado en el stream
            if err := stream.Send(order); err != nil {
                return err
            }
        }
    }
    return nil // Stream completado exitosamente
}
```

```go
// Cliente: recibir stream
stream, err := client.ListOrders(ctx, &pb.ListOrdersRequest{
    CustomerId: "customer-123",
})
for {
    order, err := stream.Recv()
    if err == io.EOF {
        break // Fin del stream
    }
    if err != nil {
        log.Fatal(err)
    }
    fmt.Printf("Pedido: %s - %.2f\n", order.OrderId, order.Total)
}
```

### 2. Client Streaming — Cliente Envía Múltiples Requests

```go
func (s *server) BulkCreateOrders(stream pb.OrderService_BulkCreateOrdersServer) error {
    var creados int
    for {
        req, err := stream.Recv()
        if err == io.EOF {
            // Cliente terminó de enviar. Responder con resumen.
            return stream.SendAndClose(&pb.BulkCreateResponse{Created: int32(creados)})
        }
        if err != nil {
            return err
        }
        // Procesar cada request del stream
        s.CreateOrder(context.Background(), req)
        creados++
    }
}
```

### 3. Bidirectional Streaming — Ambos Envían Simultáneamente

```go
func (s *server) StreamOrders(stream pb.OrderService_StreamOrdersServer) error {
    // Goroutine para enviar (eventos del servidor al cliente)
    go func() {
        ticker := time.NewTicker(2 * time.Second)
        defer ticker.Stop()
        for range ticker.C {
            if err := stream.Send(&pb.OrderEvent{
                Timestamp: time.Now().Unix(),
            }); err != nil {
                return
            }
        }
    }()

    // Loop principal para recibir (updates del cliente)
    for {
        update, err := stream.Recv()
        if err == io.EOF {
            return nil
        }
        if err != nil {
            return err
        }
        log.Printf("Update recibido: %s -> %s", update.OrderId, update.NewStatus)
    }
}
```

---

## 18.4 Interceptors — Middleware para gRPC

```go
// Interceptor unario (para RPCs normales)
func loggingInterceptor(
    ctx context.Context,
    req interface{},
    info *grpc.UnaryServerInfo,
    handler grpc.UnaryHandler,
) (interface{}, error) {
    start := time.Now()
    resp, err := handler(ctx, req)
    duration := time.Since(start)

    statusCode := codes.OK
    if err != nil {
        statusCode = status.Code(err)
    }

    log.Printf("gRPC %s | %s | %s",
        info.FullMethod,
        statusCode,
        duration,
    )
    return resp, err
}

// Interceptor de stream
func loggingStreamInterceptor(
    srv interface{},
    ss grpc.ServerStream,
    info *grpc.StreamServerInfo,
    handler grpc.StreamHandler,
) error {
    start := time.Now()
    err := handler(srv, ss)
    log.Printf("gRPC stream %s | %s", info.FullMethod, time.Since(start))
    return err
}

// Registrar interceptors
s := grpc.NewServer(
    grpc.UnaryInterceptor(loggingInterceptor),
    grpc.StreamInterceptor(loggingStreamInterceptor),
)
```

### Interceptor de Autenticación

```go
func authInterceptor(ctx context.Context) (context.Context, error) {
    md, ok := metadata.FromIncomingContext(ctx)
    if !ok {
        return nil, status.Error(codes.Unauthenticated, "metadata requerida")
    }

    tokens := md.Get("authorization")
    if len(tokens) == 0 {
        return nil, status.Error(codes.Unauthenticated, "token requerido")
    }

    // Validar JWT o token
    claims, err := validateJWT(strings.TrimPrefix(tokens[0], "Bearer "))
    if err != nil {
        return nil, status.Error(codes.Unauthenticated, "token inválido")
    }

    // Inyectar claims en el contexto
    return context.WithValue(ctx, "user_id", claims.UserID), nil
}
```

---

## 18.5 Manejo de Errores con Status Codes

gRPC tiene su propio sistema de códigos de estado, más rico que HTTP:

```go
// Códigos comunes y su equivalente HTTP
// OK (0)              → 200
// InvalidArgument (3) → 400
// NotFound (5)        → 404
// AlreadyExists (6)   → 409
// PermissionDenied (7) → 403
// Unauthenticated (16) → 401
// Internal (13)        → 500
// Unavailable (14)     → 503
// DeadlineExceeded (4) → 504

func (s *server) GetOrder(ctx context.Context, req *pb.GetOrderRequest) (*pb.Order, error) {
    if req.OrderId == "" {
        return nil, status.Error(codes.InvalidArgument, "order_id requerido")
    }

    order, ok := s.orders[req.OrderId]
    if !ok {
        return nil, status.Errorf(codes.NotFound, "pedido %s no encontrado", req.OrderId)
    }

    return order, nil
}
```

### Enriquecer Errores con Detalles

```go
import (
    "google.golang.org/genproto/googleapis/rpc/errdetails"
    "google.golang.org/grpc/status"
)

func (s *server) CreateOrder(ctx context.Context, req *pb.CreateOrderRequest) (*pb.CreateOrderResponse, error) {
    if len(req.Items) > 100 {
        st := status.New(codes.InvalidArgument, "demasiados items")
        
        // Añadir detalles estructurados al error
        st, _ = st.WithDetails(&errdetails.BadRequest{
            FieldViolations: []*errdetails.BadRequest_FieldViolation{
                {
                    Field:       "items",
                    Description: fmt.Sprintf("máximo 100 items, recibidos %d", len(req.Items)),
                },
            },
        })
        return nil, st.Err()
    }
    // ...
}
```

---

## 18.6 Cuándo Usar gRPC vs REST en Go

| Criterio | gRPC | REST |
|----------|------|------|
| **Rendimiento** | Excelente (binario, HTTP/2) | Bueno (JSON, HTTP/1.1 o 2) |
| **Streaming** | Nativo (4 tipos) | SSE o WebSocket manual |
| **Browser** | Necesita grpc-web | Nativo |
| **Debugging** | grpcurl, BloomRPC | curl, navegador |
| **Schema** | Protobuf (.proto) | OpenAPI/Swagger |
| **Generación código** | Automática, tipada | Manual o codegen |
| **Ecosistema Go** | Nativo, excelente | Nativo, excelente |

**Usa gRPC**: Comunicación service-to-service, APIs internas de alto rendimiento, streaming, ambientes polyglot.

**Usa REST**: APIs públicas, clientes web, cuando el tooling HTTP (curl, proxies, CDNs) es necesario.

---

> **Reflexión del capítulo**: gRPC + Go es la combinación más poderosa para construir APIs de alto rendimiento. La generación automática de tipos elimina categorías enteras de bugs (campos mal escritos, tipos incorrectos, contratos no cumplidos). Si tu sistema tiene más de 2 servicios que se comunican entre sí, evalúa seriamente gRPC.
