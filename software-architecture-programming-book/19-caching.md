# Capítulo 19: Estrategias de Caching

> "Hay dos cosas difíciles en computación: invalidación de caché y nombrar cosas." — Phil Karlton

## 19.1 El Principio del Caché

Guardar datos costosos de obtener en un almacenamiento más rápido y cercano. El caché es una compensación: consumes memoria/espacio a cambio de velocidad.

## 19.2 Niveles de Caché

```
Usuario ──► Browser Cache
                │
                ▼
           CDN Cache (Cloudflare, CloudFront)
                │
                ▼
           API Gateway Cache
                │
                ▼
           Application Cache (Caffeine, in-memory)
                │
                ▼
           Distributed Cache (Redis, Memcached)
                │
                ▼
           Database Cache (Buffer pool, query cache)
                │
                ▼
           Base de datos (disco)
```

Cada nivel reduce la carga del siguiente. Un hit en CDN evita que la request llegue a tu servidor.

## 19.3 Patrones de Caché

### Cache-Aside (Lazy Loading)
El más común. La aplicación controla la lógica.

```python
def get_producto(producto_id):
    # 1. Buscar en caché
    producto = cache.get(f"producto:{producto_id}")
    if producto:
        return producto  # Cache hit

    # 2. Si no está en caché, buscar en BD
    producto = db.query("SELECT * FROM productos WHERE id = ?", producto_id)

    # 3. Guardar en caché para la próxima
    if producto:
        cache.set(f"producto:{producto_id}", producto, ttl=3600)

    return producto
```

**Ventajas**: Simple, caché poblado bajo demanda.
**Desventajas**: Cache miss en primera consulta; cold start tras deploy/flush.

### Write-Through
Escribir en caché y BD simultáneamente.

```
write_through(datos):
    cache.set(key, datos)
    db.save(datos)  # Síncrono
```

**Ventajas**: Caché siempre actualizado, sin cache miss tras escritura.
**Desventajas**: Latencia de escritura (espera confirmación de BD + caché).

### Write-Behind (Write-Back)
Escribir en caché y luego asíncronamente en BD.

```
write_behind(datos):
    cache.set(key, datos)
    queue.send(db_save_event, datos)  # Asíncrono
```

**Ventajas**: Baja latencia de escritura.
**Desventajas**: Riesgo de pérdida de datos si el caché falla antes de persistir.

### Read-Through
El caché se coloca entre la app y la BD. La app solo habla con el caché.

```
App ──► Cache ◄── BD
```

**Ventajas**: Simplifica el código de aplicación.
**Desventajas**: Acoplamiento a la implementación del caché.

## 19.4 Estrategias de Invalidación

El problema más difícil del caching.

### TTL (Time to Live)
```
cache.set("producto:123", data, expire=3600)  # Expira en 1 hora
```
**Simple pero inexacto**: Datos stale durante el TTL.

### Invalidación por Eventos
Cuando los datos cambian, invalidar el caché.

```python
# Servicio de Productos
@event_handler("producto_actualizado")
def invalidar_cache_producto(evento):
    cache.delete(f"producto:{evento.producto_id}")
```

### Cache-Aside con Stale Data Tolerado
Aceptar que el caché puede tener datos ligeramente desactualizados. Definir el SLA de frescura.

## 19.5 Caché Distribuido: Redis

```
   ┌──────────┐    ┌──────────┐    ┌──────────┐
   │  App 1   │    │  App 2   │    │  App 3   │
   └────┬─────┘    └────┬─────┘    └────┬─────┘
        │               │               │
        └───────────────┼───────────────┘
                        ▼
                ┌──────────────┐
                │    Redis     │
                │  (Cluster)   │
                └──────────────┘
```

### Patrones de Uso de Redis

| Patrón | Descripción |
|--------|-------------|
| **Key-Value Store** | Caché simple: `GET/SET user:123` |
| **Sorted Sets** | Rankings, leaderboards: `ZADD/ZRANGE` |
| **Lists/Streams** | Colas de mensajes ligeras |
| **Pub/Sub** | Notificaciones en tiempo real |
| **Rate Limiting** | Contadores con TTL |
| **Distributed Locks** | Coordinación entre servicios (Redlock) |

### Redis como Caché vs Base de Datos

```conf
# Configuración para caché (datos efímeros)
maxmemory 4gb
maxmemory-policy allkeys-lru     # Evict usando LRU
save ""                          # No persistir en disco

# Configuración para datos persistentes (colas, sesiones)
maxmemory-policy noeviction      # No evictar
save 900 1                       # Persistir RDB/AOF
```

## 19.6 CDN (Content Delivery Network)

Caché geográficamente distribuida para contenido estático y edge computing.

```
Usuario en Tokio ──► Edge Tokyo (50ms) ──► Origen Virginia (200ms)
                     │    (hit = 50ms)
                     │    (miss = 50ms + 200ms)
                     ▼
                  Caché CDN
```

### Qué Cachear en CDN
- **Assets estáticos**: CSS, JS, imágenes, fuentes.
- **API responses** inmutables o semi-estáticas (páginas de producto).
- **GraphQL persisted queries** (solo queries predefinidas).

### Qué NO Cachear en CDN
- Datos específicos de usuario.
- Datos que cambian en tiempo real.
- Información sensible sin control de acceso (aunque CDNs modernos soportan autenticación en edge).

### Cache-Control Headers

```
# Browser: cachear 1 año (immutable content-hash assets)
Cache-Control: public, max-age=31536000, immutable

# CDN: cachear 5 minutos, browser: revalidar
Cache-Control: public, max-age=300, s-maxage=300, stale-while-revalidate=60

# Nunca cachear
Cache-Control: no-store, no-cache, must-revalidate, private
```

## 19.7 Anti-Patrones de Caché

| Anti-Patrón | Problema | Solución |
|-------------|----------|----------|
| **Thundering Herd** | Muchos requests simultáneos en cache miss saturan la BD | Cache stampede prevention (bloqueo en recarga) |
| **Cache como fuente de verdad** | Caché tiene datos que no están en BD | La BD es siempre la fuente de verdad |
| **Keys ilimitadas** | Sin política de evicción, Redis se queda sin memoria | TTL + maxmemory policy |
| **Caché de todo** | Caché para datos que no se consultan | Cachea solo hot data |
| **Invalidación masiva** | Limpiar todo el caché en cada deploy | Invalidación selectiva, TTLs escalonados |

## 19.8 Cuándo Cachear

**Cachea cuando**:
- Los datos se leen mucho más de lo que se escriben (>10:1).
- La latencia de origen es alta (>50ms).
- El costo computacional de generar los datos es significativo.
- Los datos son compartidos entre múltiples usuarios.

**No cachees cuando**:
- Los datos cambian constantemente.
- El volumen de datos es bajo y cabe en memoria local.
- La complejidad del caché supera sus beneficios.

---

> **Reflexión del capítulo**: El caching es una de las optimizaciones más efectivas, pero también una de las más traicioneras. La invalidación incorrecta es la fuente de bugs más difíciles de reproducir. Empieza sin caché, mide, y añade caché solo donde los datos demuestren que es necesario.
