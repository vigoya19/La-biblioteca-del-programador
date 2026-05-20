# Capítulo 31: Estrategias de Testing para Arquitectos

> "Un arquitecto que no diseña para la testabilidad está diseñando un sistema que no se puede verificar."

## 31.1 El Arquitecto y el Testing

Testing no es solo responsabilidad de los desarrolladores. El arquitecto debe:

1. **Diseñar para la testabilidad**: Si el diseño no permite testear fácilmente, el diseño es incorrecto.
2. **Definir la estrategia de testing**: Qué tipos de tests, en qué proporción, qué herramientas.
3. **Asegurar cobertura de escenarios arquitectónicos**: Los atributos de calidad necesitan tests.
4. **Diseñar ambientes de testing**: Cómo se crean, destruyen e integran en CI/CD.
5. **Establecer la cultura de calidad**: Umbrales de cobertura, definición de done, bloqueo de PRs.

## 31.2 La Pirámide de Testing

```
       ┌──────┐
       │ E2E  │  ← Pocos, lentos, frágiles. Validan flujos críticos.
       │      │     10% de los tests
      ┌┴──────┴┐
      │ Integr.│  ← Verifican contratos entre componentes reales.
      │        │     20% de los tests
     ┌┴────────┴┐
     │   Unit   │  ← Muchos, rápidos, estables. Base de la pirámide.
     │          │     70% de los tests
     └──────────┘
```

**Principio**: Si un bug puede encontrarlo un test unitario, debe ser un test unitario. Los tests de integración/E2E son más caros y frágiles; úsalos con moderación.

### La Pirámide Invertida (Anti-Patrón)

```
     ┌──────────┐
     │   Unit   │  ← Pocos. Los devs no confían en ellos.
     └──────────┘
    ┌────────────┐
    │  Integr.   │
    └────────────┘
   ┌──────────────┐
   │     E2E      │  ← Muchos. Test suite de 2 horas.
   │              │      Frágil, lenta, nadie la corre.
   └──────────────┘
```

## 31.3 Tipos de Tests que el Arquitecto Debe Exigir

### Tests Unitarios
Prueban una unidad de código aislada. Rápidos, confiables.

```typescript
describe('Order.confirm()', () => {
  it('confirma un pedido en estado PENDIENTE', () => {
    const order = new Order({ status: OrderStatus.PENDING })
    order.confirm()
    expect(order.status).toBe(OrderStatus.CONFIRMED)
  })

  it('lanza error si el pedido ya está CONFIRMADO', () => {
    const order = new Order({ status: OrderStatus.CONFIRMED })
    expect(() => order.confirm()).toThrow(OrderAlreadyConfirmedError)
  })

  it('lanza error si el pedido está CANCELADO', () => {
    const order = new Order({ status: OrderStatus.CANCELLED })
    expect(() => order.confirm()).toThrow(OrderNotConfirmableError)
  })
})
```

### Tests de Contrato (Contract Tests)
Verifican que la interacción entre dos servicios cumple el contrato pactado.

```
Servicio A (Consumidor) ─── contrato ───► Servicio B (Proveedor)

# El consumidor define sus expectativas
Pact.consumer('OrderService')
  .hasPactWith('PaymentService')
  .uponReceiving('a payment request')
    .withRequest({ method: 'POST', path: '/payments', body: {...} })
  .willRespondWith({ status: 200, body: {...} })

# El proveedor verifica que cumple el contrato
Pact.provider('PaymentService')
  .honoursPactWith('OrderService')
```

**Herramientas**: Pact, Spring Cloud Contract.

### Tests de Integración
Verifican la interacción real entre componentes (con BD real, broker real).

```typescript
describe('OrderRepository (PostgreSQL)', () => {
  // Usa TestContainers para levantar PostgreSQL real
  let container: StartedPostgreSqlContainer
  let repo: OrderRepository

  beforeAll(async () => {
    container = await new PostgreSQLContainer('postgres:16').start()
    repo = new PostgresOrderRepository(container.getConnectionUri())
    await repo.migrate()
  })

  afterAll(async () => {
    await container.stop()
  })

  it('guarda y recupera un pedido', async () => {
    const order = new Order({ id: '123', status: 'CONFIRMED' })
    await repo.save(order)
    const found = await repo.findById('123')
    expect(found.status).toBe('CONFIRMED')
  })
})
```

### Tests de Componente (In-Process)
Testean un servicio completo con dependencias externas mockeadas o en memoria. Más rápidos que E2E, más amplios que unitarios.

```
Servicio completo levantado en el mismo proceso:
- BD: H2 en memoria o TestContainers
- HTTP: MockMvc (Spring) o Supertest (Node)
- Mensajería: Embedded Kafka o mock
- Externos: WireMock
```

### Tests de Resiliencia
Prueban que el sistema maneja fallos correctamente. **Esto es responsabilidad del arquitecto asegurar.**

```java
// Test de Circuit Breaker con Resilience4j
@Test
void paymentCircuitBreakerOpensAfterFailures() {
    // Configurar mock para que falle
    given(paymentGateway.charge(any())).willThrow(new TimeoutException());

    // Ejecutar suficientes llamadas para abrir el circuito
    for (int i = 0; i < 5; i++) {
        assertThrows(TimeoutException.class, () -> paymentService.charge(order));
    }

    // Verificar que el circuito está abierto
    CircuitBreaker cb = circuitBreakerRegistry.circuitBreaker("paymentService");
    assertThat(cb.getState()).isEqualTo(CircuitBreaker.State.OPEN);

    // Verificar que llamadas subsiguientes fallan rápido
    assertThrows(CallNotPermittedException.class, () -> paymentService.charge(order));
}
```

### Tests de Carga y Estrés
Validan atributos de calidad (rendimiento, escalabilidad). (Ver Capítulo 18, Sección 18.5)

### Tests de Seguridad
- **SAST** (Static Application Security Testing): SonarQube, Checkmarx.
- **DAST** (Dynamic): OWASP ZAP, Burp Suite.
- **SCA** (Software Composition Analysis): Snyk, Dependabot, Trivy.
- **IAST** (Interactive): Combina SAST + DAST durante tests funcionales.

## 31.4 Diseñando para la Testabilidad

### Síntomas de Mala Testabilidad

| Síntoma | Causa Arquitectónica | Solución |
|---------|---------------------|----------|
| Test de 500 líneas con 20 mocks | Componente con demasiadas dependencias | Refactor: dividir responsabilidades (SRP) |
| "No puedo testear esto sin la BD real" | Lógica de negocio acoplada a persistencia | Separar dominio de infraestructura (Hexagonal) |
| Test se rompe con cada cambio sin relación | Acoplamiento excesivo entre módulos | Dependency Inversion, interfaces estables |
| Mockear el reloj para testear timeouts | Lógica temporal hardcodeada | Inyectar Clock/TimeProvider como dependencia |
| Test tarda 30 segundos en arrancar | Inicialización pesada | Componentes ligeros, lazy init |
| "Ese código es imposible de testear" | Violación de principios SOLID | Re-diseñar el componente |

### Ports & Adapters para Testabilidad

```
La arquitectura hexagonal es fundamental para la testabilidad:

Test unitario del dominio:
  Order.confirm() → sin mocks, sin BD. Rápido, confiable.

Test de integración del adaptador:
  PostgresOrderRepository con BD real (TestContainers).

Test del caso de uso:
  Mockear repositorio. Verify que el caso de uso orquesta correctamente.

Test E2E:
  Todo real. 1-2 flujos críticos.
```

### Inyección de Dependencias como Habilitador

```java
// ❌ MAL: No testeable (dependencias hardcodeadas)
public class OrderService {
    private PaymentGateway gateway = new StripeGateway(apiKey);
    private OrderRepository repo = new PostgresOrderRepository(url);
}

// ✅ BIEN: Testeable (dependencias inyectadas)
public class OrderService {
    private final PaymentGateway gateway;
    private final OrderRepository repo;

    public OrderService(PaymentGateway gateway, OrderRepository repo) {
        this.gateway = gateway;
        this.repo = repo;
    }
}

// En test: inyectar mocks/stubs/fakes
OrderService service = new OrderService(mockGateway, inMemoryRepo);
```

## 31.5 Testing en Arquitecturas Distribuidas

### Test Pyramid para Microservicios

```
Cada servicio individual:
  ┌──────┐
  │ E2E  │ (flujo cross-service crítico)
  └──────┘
  ┌──────┐
  │Integr│ (con dependencias externas)
  └──────┘
  ┌──────┐
  │ Unit │ (dominio, casos de uso)
  └──────┘

Tests cross-service:
  ┌──────────────────┐
  │ Contract Tests   │ ← Lo más importante en microservicios
  └──────────────────┘
  ┌──────────────────┐
  │ Integration/Flow │ ← 1-2 flujos críticos cross-service
  └──────────────────┘
```

### El Testing Honeycomb (para Microservicios)

```
  ┌──────────────────────┐
  │    Integration        │  ← Mayor énfasis que en monolito
  │  (contratos, APIs)    │
  └──────────────────────┘
 ┌──────────────────────────┐
 │   Unitarios (dominio)     │
 └──────────────────────────┘
┌──────────────────────────────┐
│   E2E (mínimo, solo críticos) │
└──────────────────────────────┘
```

En microservicios, los tests de integración y contrato ganan importancia porque los fallos suelen estar en las interfaces entre servicios.

### Consumer-Driven Contract Testing

```
Cada consumidor define qué espera del proveedor.
El proveedor verifica que cumple todos los contratos de sus consumidores.

Herramienta: Pact Broker
https://{pact-broker}/ → Dashboard de contratos verificados
```

## 31.6 Estrategia de Ambientes

```
┌──────────┐   ┌──────────┐   ┌──────────┐   ┌──────────┐   ┌──────────┐
│   Dev    │──►│   CI     │──►│ Staging  │──►│Canary/   │──►│   Prod   │
│ (local)  │   │(temporal)│   │(pre-prod)│   │BlueGreen │   │          │
└──────────┘   └──────────┘   └──────────┘   └──────────┘   └──────────┘
     │               │               │
  Unitarios     Integración      E2E, Perf,
               Contratos         Seguridad
```

**Principio**: Cada ambiente prueba lo que el anterior no puede. No duplicar tests entre ambientes.

### Ambientes Efímeros (Ephemeral Environments)

Por cada PR, crear un ambiente completo por 2 horas, ejecutar tests, destruir.

```
PR #456 → despliega stack completo (API + BD + Redis + Kafka)
       → ejecuta tests E2E
       → reporta resultados
       → destruye todo
       → costo: $0.15 por PR
```

**Herramientas**: Terraform + CI, Vercel Preview, Kubernetes namespaces temporales.

## 31.7 Métricas de Calidad que Importan

| Métrica | Qué Mide | Objetivo |
|---------|----------|----------|
| **Code Coverage (Línea)** | % de líneas ejecutadas en tests | >80% en dominio, >70% general |
| **Mutation Coverage** | % de mutantes detectados por tests | >75% (mejor indicador que line coverage) |
| **Test Flakiness** | % de tests que fallan aleatoriamente | <0.5% (0% ideal) |
| **Mean Time to Test** | Tiempo promedio de la suite | <5 min para unitarios, <30 min CI total |
| **Defect Escape Rate** | Bugs en prod / bugs totales | <10% |
| **Change Failure Rate** | % de deploys que requieren hotfix | <15% (DORA Elite) |

### Mutation Testing (Oro de la Cobertura)

```java
// Código original
if (order.total > 100) {
    applyDiscount(10);
}

// Mutante (el test debe fallar)
if (order.total >= 100) {  // Cambiado > por >=
    applyDiscount(10);
}

// Si el test NO falla con el mutante → el test no es bueno.
// Mutation coverage = % de mutantes que los tests detectan.
```

**Herramientas**: Pitest (Java), Stryker (JS/.NET), Mull (C++).

## 31.8 Checklist de Testing para el Arquitecto

Antes de aprobar un diseño, verifica:

- [ ] ¿El dominio puede testearse unitariamente sin BD, sin red, sin mocks pesados?
- [ ] ¿Las dependencias externas se inyectan (no se instancian con `new`)?
- [ ] ¿Los contratos entre servicios están definidos y tienen contract tests?
- [ ] ¿Los escenarios de resiliencia tienen tests automatizados?
- [ ] ¿Hay tests de carga que validen los SLOs?
- [ ] ¿El pipeline de CI/CD bloquea deploys si la cobertura baja?
- [ ] ¿Los ambientes de testing se crean/destruyen automáticamente?
- [ ] ¿El tiempo total de la suite de tests permite deploys frecuentes (<30 min)?
- [ ] ¿Los datos de prueba no contienen información real o PII?
- [ ] ¿Existe un plan para tests de seguridad (SAST, DAST, SCA)?

---

> **Reflexión del capítulo**: La testabilidad no es un accidente, es una decisión de diseño. Si un sistema es difícil de testear, es difícil de mantener, de cambiar y de confiar. El arquitecto que no diseña para el testing está diseñando para el fracaso. Los tests son la red de seguridad que permite a los equipos moverse rápido sin romper cosas. Invierte en ellos.
