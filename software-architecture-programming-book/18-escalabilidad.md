# Capítulo 18: Escalabilidad y Rendimiento

> "La escalabilidad no es manejar millones de usuarios; es mantener la experiencia cuando creces de 100 a 1,000,000."

## 18.1 Escalabilidad Vertical vs Horizontal

### Vertical (Scale Up)
Añadir más recursos a una máquina: más CPU, RAM, disco.

```
Servidor 1:            Servidor 1 (mejorado):
  2 vCPU       ──►       16 vCPU
  4 GB RAM     ──►       64 GB RAM
  100 GB SSD   ──►       500 GB NVMe
```

**Ventajas**: Simple (no cambia la arquitectura), consistencia fuerte, sin latencia de red.
**Desventajas**: Límite físico (no infinito), punto único de fallo, downtime al escalar, costoso.

### Horizontal (Scale Out)
Añadir más máquinas en paralelo.

```
Servidor 1 ──┐
Servidor 2 ──┼── Balanceador de carga ──► Usuarios
Servidor 3 ──┘
```

**Ventajas**: Escala casi infinitamente, alta disponibilidad, más económico.
**Desventajas**: Complejidad (red, consistencia, coordinación), latencia añadida.

### La Regla de Oro
> "Escala vertical hasta que duela, luego escala horizontal."

## 18.2 Estrategias de Escalabilidad Horizontal

### Stateless Services (Recomendado)
El estado está en la BD o caché, no en la instancia del servicio.

```
✅ BIEN:
Servicio A (instancia 1) → sin estado local
Servicio A (instancia 2) → sin estado local
Servicio A (instancia 3) → sin estado local
Cualquier instancia puede manejar cualquier request.

❌ MAL:
Servicio B (instancia 1) → memoria: {session: "user123"}
Servicio B (instancia 2) → memoria: {session: "user456"}
Si user123 cae en instancia 2, pierde su sesión.
```

**Solución para estado**: Externalizar sesiones a Redis, caché distribuida o sticky sessions (último recurso).

### Sharding / Particionamiento
Dividir datos entre múltiples nodos por una clave.

```
Shard 0: usuarios A-M
Shard 1: usuarios N-Z
```

**Clave de sharding**: Elíjela para distribuir uniformemente. Un sharding mal elegido crea "hot shards".

### Replicación
Copias de solo lectura de los datos.

```
┌──────────────┐
│ Escritura    │ (Primary/Master)
└──────┬───────┘
       │ replicación
   ┌───┴───┬───────┐
   ▼       ▼       ▼
Réplica  Réplica  Réplica  (solo lectura)
```

**Beneficio**: Escalar lecturas horizontalmente. Las escrituras siguen yendo al primario.

## 18.3 Métricas de Rendimiento Clave

### Latencia
Tiempo de procesamiento de una operación.

**Percentiles**: 
- p50 (mediana): La mitad de los requests son más rápidos.
- p95: 95% son más rápidos; 5% son más lentos.
- p99: Los outliers extremos.

> *"Un promedio es una mentira que esconde la verdad. Usa percentiles."*

### Throughput
Operaciones por segundo que el sistema puede manejar.

### Ley de Little
```
L = λ × W

Donde:
L = Número de requests en el sistema
λ = Tasa de llegada (throughput)
W = Tiempo promedio en el sistema (latencia)
```

Útil para dimensionar sistemas: si λ=1000 req/s y W=200ms, necesitas capacidad para L=200 requests concurrentes.

## 18.4 Cuellos de Botella Comunes

| Componente | Problema Típico | Solución |
|-----------|----------------|----------|
| **Base de datos** | Conexiones saturadas | Connection pooling, read replicas |
| **CPU** | Cálculos intensivos | Procesamiento asíncrono, workers |
| **Memoria** | Memory leaks, datos enormes | Paginación, streaming |
| **Red** | Latencia entre servicios | Co-localización, gRPC, menos hops |
| **Disco I/O** | Escrituras síncronas | Escritura asíncrona, WAL, SSD |
| **Locks** | Contención | Optimistic locking, lock-free structures |

## 18.5 Load Testing

No adivines. Mide.

### Herramientas
- **k6** (Grafana): Scripting en JS, CI/CD-friendly.
- **Artillery**: YAML-based, fácil de empezar.
- **wrk/wrk2**: HTTP benchmarking de alto rendimiento.
- **Locust**: Python, distribuido.
- **Vegeta**: CLI simple, HTTP load testing.

### Estrategia de Load Testing
1. **Baseline**: Mide el rendimiento actual.
2. **Load test**: Carga esperada normal.
3. **Stress test**: Aumenta hasta que falla → encuentra el límite.
4. **Soak test**: Carga sostenida por horas → memory leaks, degradación.
5. **Spike test**: Picos repentinos → comportamiento de auto-scaling.

## 18.6 Ley de Amdahl y Optimización

```
Speedup = 1 / ((1 - P) + P/N)

Donde:
P = Proporción paralelizable del código
N = Número de procesadores

Límite: Speedup_max = 1 / (1 - P)  (cuando N → ∞)
```

**Implicación**: Si solo el 50% de tu código es paralelizable, por más máquinas que añadas, máximo duplicas la velocidad. Optimiza primero el código secuencial.

## 18.7 Patrones Avanzados de Escalabilidad

### Database Connection Pooling
Uno de los errores más comunes al escalar: abrir una conexión por request.

```java
// Configuración de HikariCP (pool de conexiones)
HikariConfig config = new HikariConfig();
config.setMaximumPoolSize(20);              // No 1000
config.setMinimumIdle(5);
config.setConnectionTimeout(3000);          // Falla rápido, no esperes
config.setIdleTimeout(600000);
config.setMaxLifetime(1800000);
config.setLeakDetectionThreshold(10000);    // Detecta conexiones no devueltas
```

**Fórmula práctica para pool size**:
```
pool_size = (core_count * 2) + effective_spindle_count

Para una app típica con PostgreSQL:
  - 4 cores → 10-15 conexiones (no 100)

Más conexiones ≠ más throughput. Más conexiones = contención en BD.
```

### Backpressure
Cuando el productor es más rápido que el consumidor, necesitas regular el flujo.

```
Sin Backpressure:
Productor (10k msg/s) ──► Cola ──► Consumidor (1k msg/s) → Cola infinita → OOM

Con Backpressure (Reactive Streams):
Productor ──► request(N) ──► Consumidor
          ◄── N procesados ──
          ──► request(N) ──►
```

```java
// RxJava / Project Reactor: el consumidor controla el ritmo
Flux.range(1, 1000000)
    .onBackpressureBuffer(1000)  // Buffer limitado
    .subscribe(
        data -> process(data),
        error -> handleOverflow(error)
    );
```

### Throttling y Rate Shaping
Proteger el sistema regulando el tráfico entrante.

```
Token Bucket para dar forma al tráfico:

┌─────────────────────────┐
│     Token Bucket        │
│  (se llena a 100 tok/s) │
│  capacidad: 200 tokens  │
└─────────────────────────┘

Tráfico sostenido: 100 req/s → sin cola
Ráfaga corta:    200 req   → usas tokens acumulados
Ráfaga larga:    500 req/s → 300 van a cola/rechazo
```

### Consistent Hashing
Para escalar horizontalmente sistemas stateful sin redistribuir todos los datos al añadir/quitar nodos.

```
Hash Ring (0 a 2^32-1):

         0
    N3 ●   ● N1
         /\
        /  \
   N2 ●────● N4

Nodo N1 maneja keys entre N4 y N1 en el anillo.
Añadir N5 → solo N1 redistribuye ~25% de sus keys al nuevo nodo
(sin consistent hashing, redistribuirías 80% de los datos).
```

**Usos**: Sharding de caché distribuida, CDN routing, Cassandra/DynamoDB internamente.

### Shuffle Sharding
Evolución del sharding simple. Reduce el blast radius cuando un nodo falla.

```
Sharding simple (4 nodos):
  Usuario 1-25 en Nodo A
  Si Nodo A falla → 25% de usuarios afectados.

Shuffle sharding:
  Usuario 1 → Nodos [A, C, D] (elige 3 de 8)
  Usuario 2 → Nodos [B, E, G]
  Usuario 3 → Nodos [A, F, H]

  Si Nodo A falla → solo afecta usuarios que tienen A en su combinación.
  Blast radius mucho menor.
```

### Request Hedging
Para reducir latencia p99: envía el mismo request a múltiples réplicas y usa la respuesta más rápida.

```python
async def hedged_request(data):
    # Enviar a la réplica primaria
    primary = asyncio.create_task(call_service(data, "primary"))

    # Esperar p95 de latencia
    await asyncio.sleep(P95_LATENCY_MS / 1000)

    if not primary.done():
        # Hedged request: enviar a otra réplica
        hedged = asyncio.create_task(call_service(data, "secondary"))
        done, pending = await asyncio.wait(
            [primary, hedged],
            return_when=asyncio.FIRST_COMPLETED
        )
        for task in pending:
            task.cancel()
        return done.pop().result()
    else:
        return primary.result()
```

### Queue-Based Load Leveling
Usar una cola para absorber picos sin saturar el backend.

```
Usuarios ──► API Gateway ──► SQS/Kafka ──► Workers
  │              │               │              │
  │         Respuesta       Los workers       Procesan
  │         202 Accepted    consumen a su     a ritmo
  │         inmediata       propio ritmo      constante
```

## 18.8 Checklist de Escalabilidad

Antes de ir a producción con carga significativa:

- [ ] ¿Los servicios son stateless? (estado en BD o Redis, no en memoria)
- [ ] ¿Hay connection pooling configurado con límites correctos?
- [ ] ¿Todas las llamadas externas tienen timeout?
- [ ] ¿Los índices de BD cubren las queries más frecuentes?
- [ ] ¿Hay un caché multi-nivel (app → Redis → CDN)?
- [ ] ¿El auto-scaling está configurado (basado en CPU + métricas de negocio)?
- [ ] ¿Las operaciones lentas son asíncronas (colas, workers)?
- [ ] ¿Hay rate limiting en el API Gateway?
- [ ] ¿Los load tests se ejecutan semanalmente?
- [ ] ¿Los p95/p99 están dentro de los SLOs bajo carga 2x?
- [ ] ¿El plan de capacidad predice crecimiento a 6, 12 y 18 meses?
- [ ] ¿Hay un runbook de "¿qué hacer si el tráfico se triplica en 5 minutos?"
