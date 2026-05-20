# Capítulo 28: Caso de Estudio — Diseñando ShopFlow desde Cero

> "La teoría se entiende. Los casos de estudio se sienten. Los errores se recuerdan."

## 28.1 El Escenario

**Cliente**: ShopFlow, una startup que quiere competir en e-commerce en Latinoamérica.

**Requisitos Iniciales del Negocio:**
- Catálogo de 50,000 productos.
- Registro de usuarios con perfiles.
- Carrito de compras persistente.
- Procesamiento de pagos (múltiples gateways: Stripe, MercadoPago, PSE).
- Gestión de inventario en tiempo real.
- Tracking de envíos.
- Notificaciones (email, SMS, push).
- Panel de administración.
- Reportes de ventas.

**Expectativas de Crecimiento:**
- Año 1: 1,000 usuarios activos/día, ~100 pedidos/día.
- Año 2: 50,000 usuarios activos/día, Black Friday con picos de 10x.
- Año 3: Expansión a 5 países, 500,000 usuarios activos/día.

**Restricciones:**
- Time to market: 4 meses para MVP.
- Equipo: 8 desarrolladores (3 frontend, 4 backend, 1 DevOps).
- Presupuesto cloud: $5,000/mes inicial, escalando con ingresos.
- Debe cumplir PCI-DSS (manejo de pagos) y leyes de protección de datos locales.

## 28.2 Fase 1: Discovery y Análisis

### Quality Attribute Workshop

Reunimos a stakeholders (CEO, CTO, Product Manager, equipo de desarrollo) para priorizar atributos de calidad:

**Resultados del QAW:**

| Atributo | Prioridad | Justificación |
|----------|-----------|---------------|
| Time to Market | Crítico | 4 meses o mueren |
| Disponibilidad | Alta | Un minuto caídos = ventas perdidas |
| Escalabilidad | Media | Necesitan soportar picos 10x en Black Friday |
| Seguridad | Crítica | Manejan pagos y datos personales |
| Mantenibilidad | Alta | El equipo crecerá, el código debe ser entendible |
| Rendimiento | Media | Latam tiene conexiones más lentas |

### Event Storming Inicial

```
Identificamos el flujo principal de compra:

1. Usuario navega catálogo (Evento: ProductosVistos)
2. Usuario añade al carrito (Evento: ProductoAñadidoAlCarrito)
3. Usuario inicia checkout (Evento: CheckoutIniciado)
4. Usuario ingresa dirección (Evento: DireccionRegistrada)
5. Usuario selecciona método de pago (Evento: MetodoPagoSeleccionado)
6. Sistema procesa pago (Evento: PagoProcesado)
7. Sistema reserva inventario (Evento: InventarioReservado)
8. Sistema confirma pedido (Evento: PedidoConfirmado)
9. Almacén prepara envío (Evento: PedidoEnviado)
10. Usuario recibe pedido (Evento: PedidoEntregado)
```

### Bounded Contexts Identificados

```
┌──────────────────────────────────────────────────────────┐
│                    ShopFlow                               │
│                                                           │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐               │
│  │Catálogo  │  │ Usuarios  │  │ Carrito  │               │
│  │Contexto  │  │Contexto   │  │Contexto  │               │
│  └──────────┘  └──────────┘  └──────────┘               │
│                                                           │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐               │
│  │ Pedidos  │  │ Inventario│  │  Pagos   │               │
│  │Contexto  │  │Contexto   │  │Contexto  │               │
│  └──────────┘  └──────────┘  └──────────┘               │
│                                                           │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐               │
│  │  Envíos  │  │ Notificac.│  │Analytics │               │
│  │Contexto  │  │Contexto   │  │Contexto  │               │
│  └──────────┘  └──────────┘  └──────────┘               │
└──────────────────────────────────────────────────────────┘
```

## 28.3 Fase 2: Decisión Arquitectónica — ADR-001

### ADR-001: Estilo Arquitectónico Inicial

```
## Contexto
- 8 desarrolladores, 4 meses para MVP
- Expectativa de crecimiento a 3 países en 3 años
- Equipo con experiencia en Node.js y React

## Decisión: Monolito Modular
Comenzaremos con un monolito modular con PostgreSQL,
estructurado con principios de Clean Architecture.

## Justificación
- Monolito: El equipo es pequeño. La complejidad de microservicios
  mataría la velocidad del MVP.
- Modular: Separación clara por bounded contexts (paquetes/módulos)
  para facilitar extracción futura a microservicios.
- PostgreSQL: Datos altamente relacionales, transacciones ACID.
- Node.js: Experiencia del equipo, buen ecosistema.

## Estrategia de Evolución
- Fase 1 (MVP, meses 1-4): Monolito modular con PostgreSQL
- Fase 2 (Crecimiento, meses 6-12): Extraer catálogo a servicio propio
  con Elasticsearch para búsquedas.
- Fase 3 (Escala, meses 12-24): Extraer pagos, inventario y notificaciones
  como servicios independientes.
- Fase 4 (Internacional, meses 24-36): Despliegue multi-región,
  datos particionados por país.

## Consecuencias
- Positivo: Rápido desarrollo inicial, transacciones simples.
- Negativo: Si crecemos más rápido de lo esperado, el monolito dolerá antes.
- Mitigación: Mantener módulos estrictamente separados desde el día 1.
```

## 28.4 Fase 3: Diseño Detallado

### Estructura del Monolito Modular

```
shopflow/
├── packages/
│   ├── catalog/           ← Catálogo Context
│   │   ├── domain/        # Product, Category, Price (Value Objects)
│   │   ├── application/   # SearchProducts, GetProductDetails
│   │   ├── infrastructure/# PostgresCatalogRepo, S3ImageStore
│   │   └── adapter/       # CatalogController (REST)
│   │
│   ├── users/             ← Usuarios Context
│   │   ├── domain/        # User, UserProfile, Address
│   │   ├── application/   # RegisterUser, UpdateProfile
│   │   ├── infrastructure/# PostgresUserRepo, Auth0Adapter
│   │   └── adapter/       # UserController (REST)
│   │
│   ├── cart/              ← Carrito Context
│   │   ├── domain/        # Cart, CartItem
│   │   ├── application/   # AddToCart, RemoveFromCart
│   │   ├── infrastructure/# RedisCartRepo (sesiones efímeras)
│   │   └── adapter/       # CartController (REST)
│   │
│   ├── orders/            ← Pedidos Context
│   │   ├── domain/        # Order, OrderLine, OrderStatus
│   │   ├── application/   # PlaceOrder, CancelOrder
│   │   ├── infrastructure/# PostgresOrderRepo, KafkaEventPublisher
│   │   └── adapter/       # OrderController (REST)
│   │
│   ├── payments/          ← Pagos Context
│   │   ├── domain/        # Payment, PaymentMethod, Receipt
│   │   ├── application/   # ProcessPayment, RefundPayment
│   │   ├── infrastructure/# StripeAdapter, MercadoPagoAdapter
│   │   └── adapter/       # PaymentController + Webhook handlers
│   │
│   ├── inventory/         ← Inventario Context
│   │   ├── domain/        # Stock, Warehouse, Reservation
│   │   ├── application/   # ReserveStock, ReleaseStock
│   │   └── infrastructure/# PostgresInventoryRepo
│   │
│   ├── shipping/          ← Envíos Context
│   │   ├── domain/        # Shipment, TrackingNumber, Carrier
│   │   ├── application/   # CreateShipment, TrackShipment
│   │   └── infrastructure/# FedexAdapter, LocalCarrierAdapter
│   │
│   ├── notifications/     ← Notificaciones Context
│   │   ├── domain/        # Notification, Template, Channel
│   │   ├── application/   # SendNotification
│   │   └── infrastructure/# EmailAdapter, SMSAdapter, PushAdapter
│   │
│   └── shared/            ← Shared Kernel
│       ├── domain/        # BaseEntity, DomainEvent, Money, Result
│       ├── infrastructure/# Database, Cache, MessageBus abstractions
│       └── api/           # Middleware, error handling, pagination
│
├── apps/
│   ├── api/               ← Monolito API (une todos los módulos)
│   └── admin/             ← Panel de administración
│
└── infrastructure/        ← Terraform, CI/CD
```

### Modelo de Datos (PostgreSQL)

```sql
-- Esquema de Catálogo
CREATE SCHEMA catalog;

CREATE TABLE catalog.products (
    id UUID PRIMARY KEY,
    sku VARCHAR(50) UNIQUE NOT NULL,
    name VARCHAR(500) NOT NULL,
    description TEXT,
    base_price DECIMAL(10,2) NOT NULL,
    currency VARCHAR(3) DEFAULT 'USD',
    status VARCHAR(20) NOT NULL DEFAULT 'DRAFT',
    category_id UUID REFERENCES catalog.categories(id),
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_products_status ON catalog.products(status);
CREATE INDEX idx_products_category ON catalog.products(category_id);
CREATE INDEX idx_products_search ON catalog.products
    USING GIN (to_tsvector('spanish', name || ' ' || COALESCE(description, '')));

-- Esquema de Pedidos
CREATE SCHEMA orders;

CREATE TABLE orders.orders (
    id UUID PRIMARY KEY,
    user_id UUID NOT NULL,
    order_number VARCHAR(20) UNIQUE NOT NULL,
    status VARCHAR(30) NOT NULL DEFAULT 'PENDING',
    subtotal DECIMAL(12,2) NOT NULL,
    tax DECIMAL(12,2) NOT NULL,
    shipping_cost DECIMAL(10,2) NOT NULL,
    total DECIMAL(12,2) NOT NULL,
    currency VARCHAR(3) DEFAULT 'USD',
    shipping_address JSONB NOT NULL,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE orders.order_lines (
    id UUID PRIMARY KEY,
    order_id UUID NOT NULL REFERENCES orders.orders(id),
    product_id UUID NOT NULL,
    sku VARCHAR(50) NOT NULL,
    name VARCHAR(500) NOT NULL,
    quantity INTEGER NOT NULL,
    unit_price DECIMAL(10,2) NOT NULL,
    total_price DECIMAL(12,2) NOT NULL
);

-- Transacciones: asegurar consistencia entre pedido e inventario
-- (dentro del monolito, esto es trivial con BEGIN/COMMIT)
```

### Estrategia de Caché

```
┌─────────────────────────────────────────────────────┐
│ Cache Hierarchy:                                    │
│                                                     │
│ L1: CDN (CloudFront)                                │
│     → Imágenes de productos (1 año)                  │
│     → Páginas de producto públicas (5 min)           │
│                                                     │
│ L2: Application Cache (Caffeine, in-memory)         │
│     → Categorías, configuraciones (10 min)           │
│                                                     │
│ L3: Distributed Cache (Redis)                        │
│     → Sesiones de carrito (24h)                     │
│     → Datos de producto (1h)                        │
│     → Rate limiting counters                        │
│                                                     │
│ L4: Database (PostgreSQL)                           │
│     → Buffer pool (configurado a 25% RAM)           │
│     → Índices de búsqueda                           │
└─────────────────────────────────────────────────────┘
```

## 28.5 Fase 4: Decisión de Infraestructura

### ADR-002: Plataforma Cloud

```
## Decisión: AWS con ECS Fargate + RDS + ElastiCache

## Justificación
- AWS tiene presencia en Latinoamérica (sa-east-1, us-east-1).
- ECS Fargate (serverless containers): Menos operaciones que Kubernetes,
  suficiente para nuestra escala inicial.
- RDS PostgreSQL: Managed, backups automáticos, read replicas.
- ElastiCache Redis: Cache + sesiones gestionado.

## Plan de Infraestructura
```

```
Arquitectura de Infraestructura MVP:

                    ┌─────────────┐
                    │  CloudFront │  (CDN + WAF)
                    └──────┬──────┘
                           │
                    ┌──────▼──────┐
                    │  ALB        │  (Load Balancer)
                    │  + WAF      │
                    └──────┬──────┘
                           │
              ┌────────────┼────────────┐
              ▼            ▼            ▼
        ┌──────────┐ ┌──────────┐ ┌──────────┐
        │ ECS Svc  │ │ ECS Svc  │ │ ECS Svc  │
        │ (API x3) │ │ (Worker) │ │ (Admin)  │
        └────┬─────┘ └────┬─────┘ └────┬─────┘
             │            │            │
             └────────────┼────────────┘
                          │
        ┌─────────────────┼─────────────────┐
        ▼                 ▼                  ▼
  ┌──────────┐    ┌──────────┐      ┌──────────────┐
  │ RDS      │    │ ElastiC. │      │ S3 (imágenes)│
  │PostgreSQL│    │ (Redis)  │      │ + CloudFront │
  └──────────┘    └──────────┘      └──────────────┘
        │
        ▼
  ┌──────────┐
  │ Read     │ (cuando tráfico lo justifique)
  │ Replica  │
  └──────────┘
```

### Terraform (Infrastructure as Code)

```hcl
# modules/rds/main.tf
resource "aws_db_instance" "main" {
  identifier     = "shopflow-${var.environment}"
  engine         = "postgres"
  engine_version = "16.2"
  instance_class = var.environment == "prod" ? "db.r6g.large" : "db.t4g.micro"

  allocated_storage     = 100
  max_allocated_storage = 500
  storage_encrypted     = true

  db_name  = "shopflow"
  username = var.db_username
  password = random_password.db_password.result

  vpc_security_group_ids = [aws_security_group.rds.id]
  db_subnet_group_name   = aws_db_subnet_group.main.name

  backup_retention_period = 30
  backup_window          = "03:00-04:00"
  maintenance_window     = "sun:04:00-sun:05:00"

  deletion_protection = var.environment == "prod"
  skip_final_snapshot = var.environment != "prod"

  enabled_cloudwatch_logs_exports = ["postgresql", "upgrade"]

  tags = {
    Environment = var.environment
    Project     = "shopflow"
    ManagedBy   = "terraform"
  }
}
```

## 28.6 Fase 5: Implementación de Pagos (El Problema Difícil)

### ADR-003: Arquitectura de Pagos y PCI-DSS

```
## Contexto
Múltiples gateways de pago, PCI-DSS compliance.
Los datos de tarjeta NUNCA deben tocar nuestros servidores.

## Decisión: Tokenización + Iframe/Payment Element

## Solución Técnica
1. Frontend usa Stripe Elements / MercadoPago Checkout
   (iframe alojado por el gateway, nuestra app nunca ve
    números de tarjeta).

2. El gateway devuelve un token (tok_xxx).

3. Backend envía el token al gateway para procesar el pago.

4. El gateway devuelve resultado (aprobado/rechazado).

5. Backend guarda referencia (payment_intent_id, NO datos de tarjeta).

## Flujo de Pago
Cliente → [Stripe Elements iframe] → Token → Backend → Stripe API → Resultado
                                                ↓
                                         Evento: PagoProcesado
                                                ↓
                                  Servicio Pedidos → Confirma pedido
                                         +
                                  Servicio Inventario → Reserva stock
```

### Implementación del Pago (con Saga de Compensación)

```typescript
// packages/payments/application/ProcessPayment.ts
export class ProcessPaymentUseCase {
  constructor(
    private readonly paymentRepo: PaymentRepository,
    private readonly gatewayFactory: PaymentGatewayFactory,
    private readonly eventBus: EventBus,
  ) {}

  async execute(command: ProcessPaymentCommand): Promise<PaymentResult> {
    // 1. Validar
    const order = await this.orderService.getOrder(command.orderId);
    if (!order.canBePaid()) {
      return PaymentResult.invalidState(order.status);
    }

    // 2. Crear registro de pago (PENDING)
    const payment = Payment.create({
      orderId: command.orderId,
      amount: order.total,
      currency: order.currency,
      method: command.paymentMethod,
      idempotencyKey: command.idempotencyKey,
    });
    await this.paymentRepo.save(payment);

    // 3. Procesar con el gateway correspondiente
    const gateway = this.gatewayFactory.forMethod(command.paymentMethod);
    const result = await gateway.charge({
      amount: order.total,
      currency: order.currency,
      paymentMethodToken: command.token, // Token del frontend, NO tarjeta
      idempotencyKey: command.idempotencyKey,
      metadata: { orderId: command.orderId },
    });

    if (result.success) {
      // 4. Actualizar pago como completado
      payment.complete(result.transactionId);
      await this.paymentRepo.save(payment);

      // 5. Emitir evento para que Pedidos confirme
      await this.eventBus.publish(new PaymentCompletedEvent({
        orderId: command.orderId,
        paymentId: payment.id,
        transactionId: result.transactionId,
        amount: order.total,
        timestamp: new Date(),
      }));

      return PaymentResult.success(payment.id);
    } else {
      // 6. Pago rechazado
      payment.fail(result.errorCode, result.errorMessage);
      await this.paymentRepo.save(payment);

      await this.eventBus.publish(new PaymentFailedEvent({
        orderId: command.orderId,
        reason: result.errorCode,
        timestamp: new Date(),
      }));

      return PaymentResult.failed(result.errorCode);
    }
  }
}
```

```typescript
// Saga: Manejo de fallos en la creación del pedido tras pago exitoso
export class PlaceOrderSaga {
  async handle(paymentCompleted: PaymentCompletedEvent): Promise<void> {
    try {
      // Paso 1: Confirmar pedido
      await this.orderService.confirmOrder(paymentCompleted.orderId);

      // Paso 2: Reservar inventario
      await this.inventoryService.reserveItems(paymentCompleted.orderId);

      // Paso 3: Notificar
      await this.notificationService.send(
        new OrderConfirmedNotification(paymentCompleted.orderId)
      );
    } catch (error) {
      // COMPENSACIÓN: Si algo falla después del pago
      logger.error('Order fulfillment failed, initiating refund', {
        orderId: paymentCompleted.orderId,
        error: error.message,
      });

      // Reembolsar el pago
      await this.paymentService.refund(paymentCompleted.paymentId);

      // Cancelar pedido
      await this.orderService.cancelOrder(
        paymentCompleted.orderId,
        'Automatic refund due to fulfillment failure'
      );
    }
  }
}
```

## 28.7 Fase 6: Preparación para Black Friday

### ADR-004: Estrategia de Escalabilidad para Eventos de Alto Tráfico

```
## Problema
Black Friday: esperamos 10x el tráfico normal durante 48 horas.

## Decisión: Auto-scaling proactivo + degradación planificada

## Estrategia
1. Auto-scaling con step scaling policies:
   - CPU > 60%: +2 instancias
   - CPU > 75%: +4 instancias
   - CPU < 30%: -2 instancias

2. Degradación planificada (Circuit Breaker proactivo):
   - Recomendaciones personalizadas: DESACTIVADAS
   - Historial de navegación: DIFERIDO (procesar después)
   - Emails de confirmación: CAMBIADO a "enviaremos pronto"
   - Búsqueda textual: REDIRIGIDA a Elasticsearch (más ligera)

3. Preparación de infraestructura (ejecutar 1 semana antes):
   terraform apply -var="black_friday_mode=true"

4. Warm-up de caches:
   - Pre-calentar catálogo, categorías, banners.
   - Subir imágenes a CDN con anticipación.
```

### Prueba de Carga (k6)

```javascript
// load-tests/black-friday.js
import http from 'k6/http';
import { check, sleep, group } from 'k6';

export const options = {
  stages: [
    { duration: '5m',  target: 100 },   // Ramp up
    { duration: '10m', target: 1000 },  // Stay at 1000
    { duration: '5m',  target: 2000 },  // Spike to 2000
    { duration: '10m', target: 2000 },  // Stay at peak
    { duration: '5m',  target: 0 },     // Ramp down
  ],
  thresholds: {
    'http_req_duration': ['p(95)<500', 'p(99)<1000'],
    'http_req_failed': ['rate<0.01'],
    'http_reqs': ['rate>500'],
  },
};

export default function () {
  const BASE_URL = __ENV.API_URL || 'http://localhost:3000';

  group('Browse catalog', () => {
    const res = http.get(`${BASE_URL}/api/catalog/products?page=1&size=20`);
    check(res, { 'status 200': (r) => r.status === 200 });
  });

  group('Search products', () => {
    const query = ['laptop', 'celular', 'tv', 'zapatos'][Math.floor(Math.random() * 4)];
    const res = http.get(`${BASE_URL}/api/catalog/search?q=${query}`);
    check(res, { 'search successful': (r) => r.status === 200 });
  });

  group('View product detail', () => {
    const productId = Math.floor(Math.random() * 1000) + 1;
    const res = http.get(`${BASE_URL}/api/catalog/products/${productId}`);
    check(res, { 'product found': (r) => r.status === 200 || r.status === 404 });
  });

  sleep(Math.random() * 3 + 1); // Simular comportamiento de usuario
}
```

## 28.8 Fase 7: Evolución a Microservicios (Año 2)

### ADR-005: Extracción de Catálogo

```
## Contexto
El catálogo es el módulo más consultado (95% lecturas, 5% escrituras).
Necesita escalar independientemente y tener búsqueda avanzada.

## Decisión: Extraer Catálogo como microservicio con Elasticsearch

## Pasos de Migración (Strangler Fig)
1. Crear "catalog-service" independiente con Elasticsearch.
2. Sincronizar datos de PostgreSQL → Elasticsearch (CDC con Debezium).
3. Redirigir lecturas al nuevo servicio vía feature flag.
4. Redirigir también las escrituras al nuevo servicio.
5. Eliminar módulo catalog del monolito.
```

```
Arquitectura Año 2:

┌─────────────────────────────────────────────────────────┐
│                      CloudFront + WAF                    │
└───────────────────────────┬─────────────────────────────┘
                            │
┌───────────────────────────▼─────────────────────────────┐
│                      API Gateway                         │
└────┬──────────┬──────────┬──────────┬──────────────────┘
     │          │          │          │
     ▼          ▼          ▼          ▼
┌─────────┐┌─────────┐┌─────────┐┌─────────┐
│Catalog  ││ Monolito││ Payments││Inventory│
│Service  ││ (Orders,││ Service ││ Service │
│         ││ Users,  ││         ││         │
│Elastic- ││ Cart,   ││ Stripe  ││ Redis   │
│search   ││Shipping)││ MPago   ││ + DB    │
└─────────┘└─────────┘└─────────┘└─────────┘
     │          │          │          │
     └──────────┼──────────┼──────────┘
                ▼          ▼
         ┌────────────────────────┐
         │    Kafka / Event Bus   │
         └────────────┬───────────┘
                      │
         ┌────────────┼────────────┐
         ▼            ▼            ▼
   ┌──────────┐┌──────────┐┌──────────┐
   │Notificac.││Analytics ││Search    │
   │Service   ││Service   ││Indexer   │
   └──────────┘└──────────┘└──────────┘
```

## 28.9 Lecciones Aprendidas

### Lo que Funcionó Bien
1. **Monolito modular como punto de partida**: Llegamos al MVP en 3.5 meses. La separación por módulos facilitó la extracción posterior.
2. **Empezar con PostgreSQL**: Las transacciones ACID nos salvaron de incontables bugs de consistencia en pagos.
3. **ADR desde el día 1**: Cuando el equipo creció a 20 personas, los ADRs eran la única fuente de verdad sobre por qué las cosas eran como eran.
4. **Terraform para todo**: Reconstruir el entorno de staging desde cero tomaba 20 minutos.

### Lo que Haríamos Diferente
1. **Implementar distributed tracing antes**: Los primeros outages en la arquitectura distribuida fueron muy difíciles de diagnosticar. OpenTelemetry se debió configurar desde el primer microservicio.
2. **No subestimar la complejidad de Kafka**: Configurar, monitorear y operar Kafka fue más trabajo que cualquiera de los microservicios.
3. **Empezar los load tests antes**: El primer Black Friday reveló problemas de conexiones a BD que los tests unitarios nunca habrían encontrado. Ahora hacemos load tests semanales.

### Métricas del Sistema en Producción (Año 2)

```
Disponibilidad:         99.95% (SLO: 99.9%)
Latencia p95 API:      230ms
Throughput normal:     800 req/s
Throughput Black Friday: 8,500 req/s
Tamaño BD:             150 GB
Costo cloud mensual:   $12,500 (vs $4,200 iniciales)
Despliegues/día:       15-20 (entre todos los servicios)
MTTR:                  8 minutos (mediana)
```

---

> **Reflexión del capítulo**: ShopFlow no es un caso hipotético. Es una amalgama de experiencias reales en múltiples startups y empresas. Los errores que evitamos en este capítulo son errores que yo (y muchos colegas) cometimos. La moraleja: no necesitas la arquitectura perfecta el día 1, necesitas la arquitectura que te permita llegar al día 100, y luego evolucionarla.

---

← [Capítulo anterior](27-futuro.md) | [Inicio](README.md) | [Capítulo siguiente →](29-entrevistas.md)
