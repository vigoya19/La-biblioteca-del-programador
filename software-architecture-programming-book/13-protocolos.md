# Capítulo 13: Protocolos de Red y Comunicación

> "Los protocolos son los idiomas que hablan los sistemas. Conocerlos es entender cómo viaja cada bit de tus datos."

## 13.1 El Modelo OSI y TCP/IP

```
┌─────────────────────────────────────────────┐
│ Capa OSI      │ TCP/IP       │ Ejemplos      │
├───────────────┼─────────────┼────────────────┤
│ 7. Aplicación │             │ HTTP, gRPC,    │
│ 6. Presentac. │ Aplicación  │ DNS, SMTP,     │
│ 5. Sesión     │             │ WebSocket, SSH │
├───────────────┼─────────────┼────────────────┤
│ 4. Transporte │ Transporte  │ TCP, UDP       │
├───────────────┼─────────────┼────────────────┤
│ 3. Red        │ Internet    │ IP, ICMP       │
├───────────────┼─────────────┼────────────────┤
│ 2. Enlace     │ Acceso Red  │ Ethernet, WiFi │
│ 1. Física     │             │ Cables, ondas  │
└───────────────┴─────────────┴────────────────┘
```

Como arquitecto, trabajas principalmente en la **capa de aplicación** (7) y tomas decisiones sobre la **capa de transporte** (4).

## 13.2 TCP vs UDP

| | TCP | UDP |
|---|-----|-----|
| **Conexión** | Orientado a conexión (handshake) | No orientado a conexión |
| **Garantía** | Entrega garantizada y ordenada | Sin garantía de entrega ni orden |
| **Velocidad** | Más lento (ack, retransmisión) | Más rápido |
| **Control de flujo** | Sí (ventana deslizante) | No |
| **Uso típico** | HTTP, DB, email, archivos | Streaming, VoIP, gaming, DNS |

### Cuándo Elegir UDP

- Datos en tiempo real donde perder un paquete es mejor que retrasarlo.
- Broadcasting/Multicasting.
- Protocolos custom de baja latencia (gaming, IoT, videoconferencia).

### QUIC (Quick UDP Internet Connections)

Protocol de Google que corre sobre UDP y ofrece las ventajas de TCP (confiabilidad, control de congestión) sin sus desventajas (head-of-line blocking). **HTTP/3** corre sobre QUIC.

```
HTTP/1.1 y HTTP/2:
Aplicación HTTP ──► TCP ──► IP

HTTP/3:
Aplicación HTTP ──► QUIC ──► UDP ──► IP
```

## 13.3 HTTP: Evolución y Versiones

### HTTP/1.1 (1997)
- Una request por conexión TCP.
- Head-of-line blocking: la respuesta N bloquea a la N+1.
- Keep-alive reutiliza conexiones, pero las requests son secuenciales.

```
Cliente ──── req1 ────► Servidor
       ◄─── res1 ────
       ──── req2 ────►
       ◄─── res2 ────
```

### HTTP/2 (2015)
- **Multiplexación**: Múltiples requests/responses en una sola conexión TCP.
- **Server Push**: El servidor puede enviar recursos antes de que el cliente los pida.
- **Compresión de headers** (HPACK).
- **Stream prioritization**.

```
Conexión TCP única:
Stream 1: req1 → res1
Stream 2: req2 → res2  (todo simultáneo)
Stream 3: req3 → res3
```

**Problema**: Head-of-line blocking en TCP. Si un paquete TCP se pierde, todos los streams se bloquean.

### HTTP/3 (2022)
- Corre sobre QUIC (UDP), no TCP.
- Elimina el head-of-line blocking de TCP.
- Conexiones más rápidas (0-RTT para conexiones previas).
- Migración de conexión: sobrevive cambios de IP (WiFi → 4G).

**Cuando migrar a HTTP/3**: Si la latencia de red es alta o inestable (móviles, zonas con mala conectividad).

## 13.4 WebSocket

Protocolo full-duplex sobre TCP. La conexión se mantiene abierta para mensajes bidireccionales.

```
Cliente ──── HTTP Upgrade ────► Servidor
       ◄─── 101 Switching ────
       ◄═════════ WebSocket ═════════►
       (mensajes bidireccionales en tiempo real)
```

**Casos de uso**: Chats, notificaciones en tiempo real, dashboards de trading, colaboración en documentos.

**Alternativa moderna**: Server-Sent Events (SSE) si solo necesitas servidor → cliente.

## 13.5 DNS (Domain Name System)

Traduce nombres de dominio a IPs. Arquitectónicamente crítico para:
- **Descubrimiento de servicios** (service discovery).
- **Balanceo de carga** (DNS round-robin, weighted routing).
- **Failover geográfico** (Route 53 latency-based routing).
- **CDN** routing.

**Consideraciones**:
- Caching DNS (TTL): Cambios no son instantáneos.
- DNS-over-HTTPS (DoH): Privacidad y seguridad.

## 13.6 TLS/SSL y mTLS

### TLS (Transport Layer Security)
Cifrado en tránsito para HTTP (HTTPS), gRPC, WebSocket, etc.

```
Handshake TLS 1.3 (simplificado):
Cliente ──► ClientHello (cipher suites soportadas)
Servidor ──► ServerHello + Certificado
Cliente ──► Verifica certificado contra CA de confianza
         ──► Intercambio de claves (Diffie-Hellman)
         ──► Canal seguro establecido
```

### mTLS (Mutual TLS)
Ambas partes presentan certificados. El servidor también verifica al cliente.

```
Cliente ──► Certificado Cliente ──► Servidor
       ◄── Certificado Servidor ◄──
       (ambos se autentican mutuamente)
```

**Casos de uso**: Comunicación service-to-service en Service Mesh (Istio, Linkerd), zero-trust networking.

## 13.7 Protocolos Específicos que Debes Conocer

| Protocolo | Capa | Uso |
|-----------|------|-----|
| **gRPC** | 7 | RPC de alto rendimiento, streaming (Capítulo 15) |
| **AMQP** | 7 | Mensajería empresarial (RabbitMQ) |
| **MQTT** | 7 | IoT, dispositivos de bajo consumo |
| **Protobuf** | 6-7 | Serialización binaria compacta |
| **Avro** | 6-7 | Serialización con schema evolution |
| **LDAP** | 7 | Directorio de usuarios/autenticación |
| **SMTP** | 7 | Envío de correo electrónico |
| **SFTP** | 7 | Transferencia segura de archivos |

---

> **Reflexión del capítulo**: No necesitas ser experto en cada protocolo, pero sí entender sus trade-offs. La elección entre TCP y UDP, entre HTTP/2 y gRPC, entre WebSocket y SSE, es tu responsabilidad como arquitecto. El protocolo correcto puede ahorrarte meses de trabajo.
