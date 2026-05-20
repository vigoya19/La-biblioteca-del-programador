# Capítulo 12: Patrones de Resiliencia

> "Todo falla, todo el tiempo. La pregunta no es si algo fallará, sino cómo responderá tu sistema." — Werner Vogels, CTO Amazon

## 12.1 El Principio Fundamental

En sistemas distribuidos, los fallos no son excepciones: **son el estado normal de operación**. La red es inestable, los discos fallan, los servicios se saturan.

Diseña para fallar de forma predecible, no para no fallar.

## 12.2 Circuit Breaker

> "Cuando un servicio falla repetidamente, deja de llamarlo y dale tiempo para recuperarse."

Inspirado en los interruptores eléctricos. Protege al sistema de hacer llamadas que probablemente fallarán, y protege al servicio caído del tráfico que lo podría saturar al recuperarse.

### Estados del Circuit Breaker

```
         ┌─────────────┐
    ┌───►│   CERRADO    │ (operación normal)
    │    └──────┬───────┘
    │           │ umbral de fallos alcanzado
    │           ▼
    │    ┌───────────────┐
    │    │    ABIERTO    │ (rechaza llamadas inmediatamente)
    │    └──────┬────────┘
    │           │ timeout expira
    │           ▼
    │    ┌───────────────────┐
    └────│  SEMI-ABIERTO    │ (permite algunas llamadas de prueba)
         └───────────────────┘
              │ éxito → CERRADO
              │ fallo → ABIERTO
```

### Configuración Típica

```yaml
circuit_breaker:
  failure_threshold: 5        # Fallos consecutivos para abrir
  timeout: 30000              # ms antes de pasar a semi-abierto
  success_threshold: 3        # Éxitos para cerrar de nuevo
  half_open_max_requests: 2   # Máx requests en semi-abierto
```

### Implementación Conceptual

```java
public class CircuitBreaker {
    private State state = State.CLOSED;
    private int failureCount = 0;
    private long lastFailureTime;

    public <T> T call(Supplier<T> supplier) {
        if (state == State.OPEN) {
            if (System.currentTimeMillis() - lastFailureTime > timeout) {
                state = State.HALF_OPEN;
            } else {
                throw new CircuitBreakerOpenException();
            }
        }
        try {
            T result = supplier.get();
            onSuccess();
            return result;
        } catch (Exception e) {
            onFailure();
            throw e;
        }
    }

    void onSuccess() {
        state = State.CLOSED;
        failureCount = 0;
    }

    void onFailure() {
        failureCount++;
        lastFailureTime = System.currentTimeMillis();
        if (failureCount >= threshold) state = State.OPEN;
    }
}
```

**Librerías**: Resilience4j (Java), Polly (.NET), Hystrix (obsoleto pero concepto válido).

## 12.3 Retry Pattern

> "Si falló, intenta de nuevo. Pero con inteligencia."

### Estrategias de Retry

| Estrategia | Descripción | Cuándo Usar |
|-----------|-------------|-------------|
| **Fixed** | Reintentar cada N ms | Fallos predecibles |
| **Exponential Backoff** | Esperar 1s, 2s, 4s, 8s... | Evitar saturar servicio caído |
| **Exponential Backoff + Jitter** | Añadir aleatoriedad al backoff | Evitar thundering herd |
| **Linear** | Esperar N, 2N, 3N... | Balance entre fixed y exponencial |

```python
# Exponential Backoff con Jitter
def retry_with_backoff(func, max_retries=3):
    for attempt in range(max_retries):
        try:
            return func()
        except RetryableException:
            if attempt == max_retries - 1:
                raise
            wait = min(2 ** attempt, 30)  # Cap a 30s
            jitter = random.uniform(0, wait * 0.5)
            time.sleep(wait + jitter)
```

**Advertencia**: No reintentes operaciones no idempotentes (crear un pedido, cobrar un pago) sin diseñar para ello.

## 12.4 Bulkhead Pattern

> "Un fallo en un compartimento no hunde todo el barco."

Aísla recursos para que un problema en una parte del sistema no afecte a otras.

### Tipos de Bulkhead

**Semaphore Bulkhead**: Limita el número de llamadas concurrentes a un servicio.

```
Servicio A ──► [Max 10 concurrentes] ──► Servicio B
Servicio A ──► [Max 5 concurrentes]  ──► Servicio C
```

Si B se satura, los threads de A no se agotan esperando a B.

**Thread Pool Bulkhead**: Pools de threads separados por servicio.

```
Thread Pool A (20 threads) ──► Servicio B
Thread Pool C (10 threads) ──► Servicio C
```

Si B se cae, el pool A se llena, pero el pool C sigue funcionando.

## 12.5 Timeout Pattern

> "No esperes para siempre. Cada recurso tiene un límite de tiempo."

Todo request externo debe tener timeout. Sin timeout, un servicio lento puede agotar todos tus threads.

```
Timeout en cascada:
Servicio A ── timeout 2s ──► Servicio B ── timeout 1s ──► Servicio C

Regla general: timeout_total > sum(timeouts_internos)
```

**Buenas prácticas**:
- Timeout de conexión: 500ms-2s
- Timeout de lectura: 2s-10s (según operación)
- Timeout global de request: incluye todos los timeouts internos + buffer

## 12.6 Fallback Pattern

> "Cuando todo falla, degrada elegantemente."

```java
@CircuitBreaker(name = "recomendaciones", fallbackMethod = "recomendacionesFallback")
public List<Producto> getRecomendaciones(Usuario u) {
    return servicioRecomendaciones.get(u);
}

public List<Producto> recomendacionesFallback(Usuario u, Exception e) {
    log.warn("Servicio de recomendaciones caído, usando caché");
    return cacheRecomendaciones.get(u.getId());
    // O: return productosMasVendidos();
    // O: return Collections.emptyList(); // Degradación cero
}
```

**Niveles de Degradación**:
1. **Caché**: Datos frescos no disponibles, pero tengo datos algo viejos.
2. **Valores por defecto**: Respuesta genérica (productos populares).
3. **Funcionalidad reducida**: El sistema funciona sin esa feature.
4. **Error controlado**: Mensaje "intenta más tarde".

## 12.7 Rate Limiter

> "Protege tus recursos del abuso y los picos de tráfico."

### Algoritmos

| Algoritmo | Descripción |
|-----------|-------------|
| **Token Bucket** | Tokens se generan a ritmo constante. Cada request consume un token. Permite ráfagas. |
| **Leaky Bucket** | Cola FIFO de tamaño fijo. Procesa a ritmo constante. No permite ráfagas. |
| **Fixed Window** | Contador por ventana de tiempo (ej: 100 req/min). Problema: ráfagas en bordes. |
| **Sliding Window Log** | Ventana deslizante con registro de timestamps. Preciso pero usa memoria. |

```java
// Rate limit en API Gateway (Kong, Nginx, Envoy)
// o en aplicación con Resilience4j
@RateLimiter(name = "apiBasico", fallbackMethod = "rateLimitFallback")
public Response procesar(Request r) { ... }

public Response rateLimitFallback(Request r, Exception e) {
    return Response.status(429).body("Demasiadas solicitudes. Intenta en 60s");
}
```

## 12.8 Health Check & Liveness/Readiness Probes

```
Liveness Probe:  ¿Está vivo el proceso?
                 Si falla → Kubernetes reinicia el pod.

Readiness Probe: ¿Puede recibir tráfico?
                 Si falla → Kubernetes lo saca del balanceador.
```

---

> **Reflexión del capítulo**: La resiliencia no es un feature, es un atributo sistémico. No puedes añadirla al final. Debes diseñarla desde el primer día, porque el día que la necesites ya será demasiado tarde.
