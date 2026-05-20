# Capítulo 29: Escenarios de Arquitectura — Entrevistas y Desafíos Reales

> "En una entrevista de arquitectura no buscan respuestas correctas, buscan cómo piensas."

## 29.1 La Mentalidad Correcta para Entrevistas de Arquitectura

Las entrevistas de system design no evalúan que memorices la arquitectura de Netflix. Evalúan:

1. **Cómo abordas problemas ambiguos** — No hay una respuesta correcta.
2. **Cómo priorizas trade-offs** — Consistencia vs disponibilidad, simplicidad vs escalabilidad.
3. **Tu profundidad técnica** — ¿Entiendes realmente las tecnologías que mencionas?
4. **Tu capacidad de comunicación** — ¿Puedes explicar decisiones complejas claramente?
5. **Cómo manejas la incertidumbre** — Preguntas clarificadoras antes de diseñar.

### El Framework para Responder

```
PASO 1 — Requisitos (3-5 min): Pregunta, no asumas.
  → ¿Usuarios? ¿Tráfico? ¿Latencia esperada? ¿Disponibilidad?

PASO 2 — Estimaciones de Back-of-the-Envelope (3-5 min):
  → Requests por segundo, almacenamiento, ancho de banda.

PASO 3 — Diseño de Alto Nivel (10-15 min):
  → Diagrama de componentes principales, APIs, flujo de datos.

PASO 4 — Deep Dive (10-15 min):
  → El entrevistador elige 2-3 áreas para profundizar.

PASO 5 — Cuellos de botella y Mejoras (5 min):
  → ¿Qué falla? ¿Cómo escalar? ¿Qué monitorear?
```

## 29.2 Escenario 1: Diseña un Acortador de URLs (TinyURL)

### Requisitos

```
Funcionales:
- Acortar una URL larga a una corta (8 caracteres).
- Redirigir de la URL corta a la original.
- Las URLs no expiran (o expiran tras N años).

No Funcionales:
- Alta disponibilidad (99.9%+).
- Baja latencia en redirecciones (<50ms p95).
- 100 millones de URLs creadas por mes.
- 1 billón de redirecciones por mes (ratio 10:1 lectura/escritura).
```

### Back-of-the-Envelope

```
Escrituras: 100M / mes = ~38 escrituras/segundo
Lecturas:   1B / mes  = ~385 lecturas/segundo
Almacenamiento: 100M * 500 bytes (URL+metadata) = 50 GB/mes
                → 600 GB/año → 3 TB en 5 años
```

### Diseño de Alto Nivel

```
┌──────────┐      ┌──────────────┐      ┌──────────┐
│ Usuario  │─────►│ API Gateway  │─────►│URL Service│
└──────────┘      └──────────────┘      └─────┬────┘
                                              │
                          ┌───────────────────┼───────────────────┐
                          ▼                   ▼                   ▼
                   ┌──────────┐       ┌──────────┐       ┌──────────┐
                   │PostgreSQL│       │  Redis   │       │   Kafka  │
                   │(fuente de│       │  (caché  │       │(analytics│
                   │ verdad)  │       │   L1)    │       │ eventos) │
                   └──────────┘       └──────────┘       └──────────┘
```

### Generación de Claves Cortas

```python
import hashlib, base64

def generate_short_key(long_url: str, user_id: str) -> str:
    """
    Estrategia: Hash + Base62
    MD5(long_url + user_id + timestamp) → 128 bits
    Tomar primeros 48 bits → 6 caracteres Base62
    Si colisión: añadir contador y reintentar
    """
    hash_input = f"{long_url}:{user_id}:{time.time_ns()}"
    hash_bytes = hashlib.md5(hash_input.encode()).digest()
    # Tomar 6 bytes (48 bits), codificar en Base62
    key = base62_encode(int.from_bytes(hash_bytes[:6], 'big'))
    return key[:8]
```

### Deep Dive: Caché y Redirección

```python
# Servicio de Redirección — Camino Feliz
async def redirect(short_key: str):
    # 1. Buscar en caché L1 (Redis) — 1ms
    cached = await redis.get(f"url:{short_key}")
    if cached:
        return RedirectResponse(url=cached['long_url'], status_code=301)

    # 2. Cache miss → Buscar en BD — 5ms
    url_record = await db.query(
        "SELECT long_url FROM urls WHERE short_key = $1", short_key
    )
    if not url_record:
        raise HTTPException(status_code=404)

    # 3. Guardar en caché
    await redis.setex(f"url:{short_key}", 86400, url_record['long_url'])

    return RedirectResponse(url=url_record['long_url'], status_code=301)

# Estrategia de caché:
# - 80% de las URLs nunca se visitan → No llenar caché con basura
# - Cachear solo en READ (no en WRITE)
# - Política LRU con TTL de 24h
```

### Problemas y Soluciones

| Problema | Solución |
|----------|----------|
| **Colisiones de hash** | Reintentar con contador incremental |
| **Hot URLs** (URLs virales) | CDN en front, caché multi-nivel |
| **Abuso/Spam** | Rate limiting por IP/API key |
| **Base de datos gigante** | Partitioning por rango de fecha |
| **Usuarios maliciosos** | Validación de URLs destino + blocklist |

---

## 29.3 Escenario 2: Diseña un Sistema de Chat en Tiempo Real (WhatsApp)

### Requisitos

```
Funcionales:
- Chat 1-a-1 y grupos (hasta 256 miembros).
- Mensajes de texto, imágenes, video.
- Estado en línea / última conexión.
- Confirmaciones de entrega y lectura (✓, ✓✓).
- Historial de mensajes.

No Funcionales:
- Baja latencia (<100ms entrega de mensaje).
- Alta disponibilidad (99.99%).
- 500M usuarios activos, 100B mensajes/día.
- Consistencia eventual aceptable.
```

### Back-of-the-Envelope

```
Usuarios activos: 500M
Mensajes/día: 100B → ~1.15M mensajes/segundo
Tamaño promedio mensaje texto: 100 bytes
Almacenamiento diario: 100B * 100B = 10 TB/día (solo texto)
                          + multimedia (más pesado)
```

### Diseño de Alto Nivel

```
┌──────────────────────────────────────────────────────────┐
│                     Cliente (App Móvil)                   │
│  ┌──────────────────────────────────────────────────┐   │
│  │ WebSocket connection al chat server               │   │
│  └──────────────────────────────────────────────────┘   │
└───────────────────────────┬──────────────────────────────┘
                            │
              ┌─────────────▼─────────────┐
              │    WebSocket Gateway       │
              │ (stateful, sticky sessions)│
              └─────────────┬─────────────┘
                            │
       ┌────────────────────┼────────────────────┐
       ▼                    ▼                    ▼
┌──────────────┐   ┌──────────────┐   ┌──────────────┐
│ Chat Service │   │ Group Service│   │ Media Service│
│ (mensajería) │   │ (grupos)     │   │ (upload S3)  │
└──────┬───────┘   └──────────────┘   └──────────────┘
       │
       ▼
┌──────────────┐
│   Kafka      │  ← Topic por chat_id (particionado)
│  (message    │     Ventaja: orden garantizado por partición
│   broker)    │     Los mensajes persisten en el log
└──────────────┘
       │
       ▼
┌──────────────┐
│  Cassandra   │  ← Datos particionados por chat_id
│  (message    │     Escalabilidad horizontal nativa
│   store)     │     Escrituras rápidas, TTL para expiración
└──────────────┘
```

### Deep Dive: Entrega de Mensajes

```python
# Flujo de envío de mensaje
async def send_message(sender_id: str, chat_id: str, content: str):
    # 1. Validar que sender pertenece al chat
    if not await group_service.is_member(chat_id, sender_id):
        raise NotMemberError()

    # 2. Persistir mensaje
    message = Message(
        id=generate_id(),
        chat_id=chat_id,
        sender_id=sender_id,
        content=content,
        timestamp=utc_now(),
    )

    # 3. Guardar en BD + publicar a Kafka (Transactional Outbox si necesario)
    await message_store.save(message)
    await kafka.publish(f"chat.{chat_id}", message)

    return message.id


# Consumidor: entrega a destinatarios
async def deliver_message(message: Message):
    # 1. Obtener miembros del chat/conversación
    members = await group_service.get_members(message.chat_id)

    # 2. Para cada miembro (excepto sender), enviar por WebSocket
    for member_id in members:
        if member_id == message.sender_id:
            continue

        # Buscar la conexión WebSocket del usuario
        connection = connection_registry.get(member_id)
        if connection and connection.is_alive():
            await connection.send(message)
            # Marcar como entregado (✓)
        else:
            # Usuario offline → la app recibirá al reconectarse
            # vía sincronización desde el último mensaje recibido
            pass
```

### Manejo de Usuarios Offline

```python
# Sincronización al reconectar
async def sync_messages(user_id: str, last_seq_id: int):
    """
    Cuando un usuario se reconecta, solicita mensajes
    desde el último sequence_id que recibió.
    """
    messages = await message_store.get_messages_since(
        user_id=user_id,
        since_seq_id=last_seq_id,
        limit=100,
    )
    return messages

# Cada mensaje tiene un server_sequence_id incremental
# El cliente guarda el último seq_id procesado
# Al reconectar: GET /sync?since={last_seq_id}
```

---

## 29.4 Escenario 3: Diseña una Plataforma de Streaming de Video (YouTube)

### Requisitos

```
- Subida de videos (hasta 10 GB, cualquier formato).
- Transcodificación a múltiples resoluciones (144p a 4K).
- Streaming adaptativo (HLS/DASH).
- 500M videos, 5B reproducciones/día.
- Recomendaciones personalizadas.
- Comentarios, likes.
```

### Diseño de Alto Nivel

```
┌──────────┐     ┌──────────────┐     ┌─────────────────┐
│ Usuario  │────►│ Upload Svc   │────►│ S3 (original)   │
└──────────┘     └──────┬───────┘     └─────────────────┘
                        │
                  ┌─────▼──────┐
                  │  Kafka /   │  ← Evento: VideoSubido
                  │  SQS       │
                  └─────┬──────┘
                        │
                  ┌─────▼──────────────────────┐
                  │  Transcoding Pipeline       │
                  │  (AWS Elemental MediaConvert│
                  │   o FFmpeg en workers)      │
                  │                             │
                  │  Input: 1080p original      │
                  │  Output: 144p, 360p, 720p,  │
                  │          1080p, 4K          │
                  │  Format: HLS (.m3u8 + .ts)  │
                  └─────┬──────────────────────┘
                        │
                  ┌─────▼──────┐
                  │  CDN       │  ← CloudFront / CloudFlare
                  │  (edge)    │     Videos cacheados en edges
                  └────────────┘     mundiales
```

### Deep Dive: Streaming Adaptativo

```
HLS (HTTP Live Streaming):
- El video se divide en segmentos de 2-10 segundos (.ts).
- Un playlist (.m3u8) lista los segmentos.
- El cliente elige la calidad según su ancho de banda.

Estructura de archivos en CDN:
/videos/{video_id}/
  ├── master.m3u8           ← Playlist maestro (lista de variantes)
  ├── 360p/
  │   ├── playlist.m3u8
  │   ├── segment_000.ts
  │   ├── segment_001.ts
  │   └── ...
  ├── 720p/
  └── 1080p/
```

### Recomendaciones (Alto Nivel)

```
┌─────────────────────────────────────────────┐
│ Pipeline de Recomendaciones:                │
│                                              │
│ 1. Recolección de eventos                   │
│    (vistas, likes, tiempo de visualización) │
│    ↓                                        │
│ 2. Feature Engineering                      │
│    (embeddings de videos, perfil usuario)   │
│    ↓                                        │
│ 3. Candidate Generation                     │
│    (filtrado colaborativo + contenido)      │
│    ↓                                        │
│ 4. Ranking                                  │
│    (modelo de ML que predice engagement)    │
│    ↓                                        │
│ 5. Re-ranking + Filtros                     │
│    (diversidad, frescura, ya vistos)        │
└─────────────────────────────────────────────┘
```

---

## 29.5 Escenario 4: Diseña un Rate Limiter Distribuido

### Requisitos

```
- Limitar requests por usuario/clave API.
- Soporte para múltiples ventanas (por segundo, minuto, hora).
- Distribuido (múltiples instancias de API Gateway).
- Baja latencia (<5ms de overhead).
- Alta precisión (no rechazar requests válidos, no permitir excesos).
```

### Algoritmos de Rate Limiting

```
1. Token Bucket:
   - Un bucket se llena de tokens a ritmo constante.
   - Cada request consume un token.
   - Permite ráfagas (si hay tokens acumulados).

2. Sliding Window Log:
   - Registro de timestamp de cada request.
   - Ventana deslizante: cuento requests en los últimos N segundos.
   - + Preciso
   - - Mucha memoria (guardar cada timestamp)

3. Sliding Window Counter (Híbrido recomendado):
   - Combina Fixed Window + peso de ventana anterior.
   - Precisión ~99% con bajo consumo de memoria.
```

### Implementación con Redis

```python
# Sliding Window Counter en Redis (Lua script para atomicidad)
RATE_LIMITER_SCRIPT = """
local key = KEYS[1]
local now = tonumber(ARGV[1])
local window = tonumber(ARGV[2])  -- en segundos
local limit = tonumber(ARGV[3])

-- Limpiar entradas antiguas
redis.call('ZREMRANGEBYSCORE', key, 0, now - window)

-- Contar requests en la ventana actual
local current = redis.call('ZCARD', key)

if current < limit then
    -- Añadir este request
    redis.call('ZADD', key, now, now .. '-' .. math.random())
    redis.call('EXPIRE', key, window)
    return {1, limit - current - 1}  -- Permitido, restantes
else
    return {0, 0}  -- Rate limit excedido
end
"""

async def check_rate_limit(user_id: str, limit: int, window_sec: int):
    key = f"rate_limit:{user_id}"
    result = await redis.eval(
        RATE_LIMITER_SCRIPT,
        keys=[key],
        args=[time.time(), window_sec, limit]
    )
    return {"allowed": result[0] == 1, "remaining": result[1]}
```

### Arquitectura del Rate Limiter

```
┌──────────┐     ┌──────────────────┐     ┌──────────┐
│ Cliente  │────►│ API Gateway      │────►│ Servicio │
│          │     │                  │     │          │
│          │     │ ┌──────────────┐ │     │          │
│          │     │ │Rate Limiter   │ │     │          │
│          │     │ │Middleware     │ │     │          │
│          │     │ └──────┬───────┘ │     │          │
│          │     │        │         │     │          │
│          │     │   ┌────▼────┐   │     │          │
│          │     │   │  Redis  │   │     │          │
│          │     │   │ Cluster │   │     │          │
│          │     │   └─────────┘   │     │          │
└──────────┘     └──────────────────┘     └──────────┘

Si Redis falla temporalmente → Fail open (permitir requests)
  Configurable según criticidad del endpoint.
```

---

## 29.6 Consejos para Entrevistas de Arquitectura

### Lo que DEBES Hacer
1. **Hacer preguntas**: Demuestra que entiendes que el contexto importa.
2. **Pensar en voz alta**: El proceso es más importante que la respuesta.
3. **Empezar simple y escalar**: Monolito → caché → colas → microservicios.
4. **Mencionar trade-offs explícitamente**: "Esto mejora X pero empeora Y."
5. **Hacer cálculos de servilleta**: Requests/segundo, almacenamiento, ancho de banda.
6. **Dibujar**: Un diagrama vale más que mil palabras (usa boxes-and-arrows).

### Lo que NO Debes Hacer
1. **Saltar a microservicios**: "Voy a usar 50 microservicios con Kubernetes" sin justificar.
2. **Mencionar tecnologías que no entiendes**: Te preguntarán sobre ellas.
3. **Ignorar los bordes**: No solo pienses en el happy path.
4. **Optimizar prematuramente**: Resuelve el problema, luego optimiza.
5. **Monologar**: Haz pausas, pregunta si vas por buen camino.

### Cómo Practicar

```
1. Estudia sistemas reales:
   - Blog de Ingeniería de Netflix, Uber, Meta, Stripe.
   - InfoQ, High Scalability.

2. Practica con otros:
   - interviewing.io (entrevistas de práctica)
   - Pramp, Exponent (peer mock interviews)

3. Graba tus sesiones de práctica:
   - Revisa tu claridad, tu uso de diagramas, tus pausas.

4. Lee papers:
   - Dynamo (Amazon), Bigtable (Google), Kafka (LinkedIn).
   - Entender los trade-offs reales que hicieron.
```

---

> **Reflexión del capítulo**: Las entrevistas de arquitectura premian la profundidad sobre la amplitud. Prefieren que conozcas 3 tecnologías en profundidad que 20 superficialmente. Cuando digas "usaría Kafka", prepárate para explicar particiones, consumer groups, exactly-once semantics y cuándo NO usarías Kafka.

---

← [Capítulo anterior](28-caso-estudio.md) | [Inicio](README.md) | [Capítulo siguiente →](30-documentacion.md)
