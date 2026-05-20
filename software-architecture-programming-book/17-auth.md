# Capítulo 17: Autenticación y Autorización — El Mini-Libro de Auth que Todo Arquitecto Necesita

> "Autenticación es quién eres. Autorización es qué puedes hacer. No las confundas jamás, porque confundirlas en producción cuesta millones."

## 17.1 La Distinción Más Importante de Tu Carrera

He visto este error en entrevistas, en diseños de sistemas, en código en producción, en arquitectos senior con 15 años de experiencia. Es el error de seguridad más común y más peligroso: **usar autenticación cuando se necesita autorización, o viceversa.**

```
┌──────────────────────────────────────────────────────────────┐
│                                                               │
│  AUTENTICACIÓN (AuthN)             AUTORIZACIÓN (AuthZ)      │
│  ─────────────────────             ────────────────────      │
│                                                               │
│  Pregunta: ¿Quién eres?            Pregunta: ¿Puedes hacer   │
│                                          esto?               │
│                                                               │
│  Análogo: Mostrar tu pasaporte     Análogo: Tener un pase    │
│           en el aeropuerto.               VIP para la sala    │
│           "Soy Juan Pérez".               de primera clase.   │
│                                           "Puedes entrar."   │
│                                                               │
│  Métodos:                         Métodos:                   │
│   • Password                      • RBAC (Roles)              │
│   • Biometría                     • ABAC (Atributos)          │
│   • OTP / MFA                     • PBAC (Políticas)         │
│   • Certificados                  • ACLs (Listas de acceso)  │
│   • Social Login                  • Scopes / Permisos        │
│                                                               │
│  Sucede PRIMERO                   Sucede DESPUÉS de authN    │
│                                                               │
│  Produce: Identidad               Produce: Decisión de acceso│
│  (subject, user_id)               (allow / deny)             │
│                                                               │
└──────────────────────────────────────────────────────────────┘
```

### La Diferencia en Código — Por Qué Confundirlas Es Mortal

```python
# ❌ EL ERROR CLÁSICO: Confundir autenticación con autorización

@app.route('/api/admin/users')
def get_all_users():
    token = request.headers.get('Authorization')
    
    # "Verifico el token" → autenticación
    user = verify_jwt(token)
    if not user:
        return {'error': 'No autenticado'}, 401
    
    # "Devuelvo todos los usuarios" → ¿DÓNDE ESTÁ LA AUTORIZACIÓN?
    # CUALQUIER usuario autenticado puede ver TODOS los usuarios.
    # El pasante que entró hoy puede descargar la BD completa de usuarios.
    users = db.query("SELECT * FROM users")
    return {'users': users}


# ✅ CORRECTO: Autenticación + Autorización separadas

@app.route('/api/admin/users')
@require_authentication  # AuthN: ¿Quién eres?
def get_all_users(current_user):
    # AuthZ: ¿Puedes hacer esto?
    if not current_user.has_permission('users:read_all'):
        return {'error': 'No autorizado'}, 403
    
    # Solo admins llegan aquí
    users = user_service.get_all_users()
    return {'users': users}
```

**La diferencia en una frase**: Autenticación te dice QUE eres Juan. Autorización te dice si Juan PUEDE borrar la base de datos. Juan puede estar perfectamente autenticado y aún así no tener permiso para ciertas operaciones.

---

## 17.2 JWT (JSON Web Tokens) — Entendido a Fondo

### ¿Qué Es un JWT Realmente?

Un JWT es un token compacto, auto-contenido y firmado digitalmente (NO encriptado por defecto) que transmite "claims" (afirmaciones) entre partes.

```
Estructura de un JWT:

eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkpvaG4gRG9lIiwicm9sZSI6IkFETUlOIiwiaWF0IjoxNTE2MjM5MDIyfQ.4GzQJ7xR2qF8kL3mN5pQrS6tUvW8xYzA1bC2dE3fG4hI

Parte 1 (Header):    eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9
  → Base64 de: {"alg": "HS256", "typ": "JWT"}
  → Dice: "Esto es un JWT, firmado con HMAC-SHA256"

Parte 2 (Payload):   eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkpvaG4gRG9lIiwicm9sZSI6IkFETUlOIiwiaWF0IjoxNTE2MjM5MDIyfQ
  → Base64 de: {"sub": "1234567890", "name": "John Doe", "role": "ADMIN", "iat": 1516239022}
  → Dice: "El usuario es John Doe, ID 1234567890, rol ADMIN"

Parte 3 (Signature): 4GzQJ7xR2qF8kL3mN5pQrS6tUvW8xYzA1bC2dE3fG4hI
  → HMAC-SHA256(Header + "." + Payload, SECRET_KEY)
  → Dice: "Este token FUE EMITIDO por alguien que conoce el SECRET_KEY"
```

### Lo Que TODO el Mundo Entiende Mal Sobre JWT

**"JWT está encriptado"** → FALSO. JWT está FIRMADO, no encriptado. El payload es Base64 (legible por cualquiera). **NUNCA pongas información sensible en un JWT.** Si necesitas encriptar el payload, usa JWE (JSON Web Encryption), no JWT estándar.

```python
import base64
import json

# CUALQUIERA puede leer el payload de tu JWT
jwt = "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiIxMjM0NTY3ODkwIiwicm9sZSI6IkFETUlOIiwiaWF0IjoxNTE2MjM5MDIyfQ.4GzQJ7xR2qF8kL3mN5pQrS6tUvW8xYzA1bC2dE3fG4hI"

payload_base64 = jwt.split('.')[1]

# Agregar padding si es necesario
payload_base64 += '=' * (4 - len(payload_base64) % 4)

# Decodificar → CUALQUIERA PUEDE LEER ESTO sin conocer el secret
payload = json.loads(base64.b64decode(payload_base64))
print(payload)
# {"sub": "1234567890", "role": "ADMIN", "iat": 1516239022}

# Lo que NO puedes hacer sin el secret es FIRMAR un token nuevo.
# El servidor verifica la firma para saber que ÉL emitió ese token.
# Pero el CONTENIDO es público. Base64 no es encriptación.
```

### Claims Estándar que Debes Usar SIEMPRE

```json
{
  // ─── Claims Registrados (RFC 7519) ───
  
  "iss": "https://auth.miempresa.com",     
  // Issuer: ¿QUIÉN emitió este token? 
  // SIEMPRE verifica que el issuer es quien esperas.
  // Previene que aceptes tokens de otro auth server.

  "sub": "user_ck4j8s2k30001x2v5q9r6h8b", 
  // Subject: ¿DE QUIÉN es este token?
  // El ID único del usuario. Lo usarás en TODAS las queries.
  // Usa UUIDs/ULIDs, no IDs secuenciales.

  "aud": "shopflow-api",                   
  // Audience: ¿PARA QUIÉN es este token?
  // Si tu API se llama "shopflow-api", rechaza tokens con aud diferente.
  // Previene que un token emitido para el servicio de analytics
  // sea usado contra el servicio de pagos.

  "exp": 1705392000,                       
  // Expiration: timestamp Unix. Después de esto, el token ES INVÁLIDO.
  // SIEMPRE verifica exp. Es tu primera línea de defensa
  // contra tokens robados (el atacante tiene ventana limitada).

  "nbf": 1705305600,                       
  // Not Before: timestamp Unix. Antes de esto, el token NO ES VÁLIDO.
  // Útil para tokens emitidos "para usar mañana".

  "iat": 1705305600,                       
  // Issued At: timestamp Unix. ¿Cuándo se emitió?
  // Útil para rechazar tokens "demasiado viejos" aunque no hayan expirado.

  "jti": "unique-token-id-abc123",         
  // JWT ID: Identificador ÚNICO de este token específico.
  // CRÍTICO para listas de revocación: "el token con jti='abc123'
  // fue revocado aunque no ha expirado".
  
  // ─── Claims Personalizados ───
  
  "role": "ADMIN",                         
  // Rol del usuario. CUIDADO: si el rol cambia en BD,
  // el JWT viejo sigue diciendo "ADMIN" hasta que expire.
  // Por eso los tokens deben tener vida corta.

  "permissions": ["orders:read", "orders:write"],
  // Permisos específicos. Más granular que "role".
  // Pero ojo: si los permisos cambian, el JWT está desactualizado.

  "tenant_id": "tenant_mx_001"
  // En sistemas multi-tenant, a qué tenant pertenece.
}
```

### El Dilema del Tiempo de Vida del JWT

```
Token con expiración LARGA (24 horas):
  ✓ Menos refrescos, mejor UX.
  ✗ Si te roban el token, el atacante tiene 24 horas.
  ✗ Si el usuario cambia su rol/permisos, el token viejo
    sigue teniendo los viejos permisos por 24 horas.
  ✗ No puedes "cerrar sesión" efectivamente (el token sigue vivo).

Token con expiración CORTA (15 minutos):
  ✓ Si te roban el token, el atacante tiene 15 minutos.
  ✓ Cambios de permisos se reflejan en 15 minutos.
  ✓ "Cerrar sesión" toma efecto en ≤15 minutos.
  ✗ Necesitas refresh tokens.
  ✗ Más complejidad de implementación.

La solución de la industria:
  Access Token:  15-30 minutos (contiene permisos, se envía en cada request).
  Refresh Token: 7-30 días (solo sirve para obtener nuevos access tokens).
                 Se almacena en HttpOnly cookie o secure storage.
                 Se puede revocar (el auth server tiene lista de refresh tokens).
```

### La Arquitectura JWT Completa

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                  │
│  1. LOGIN                                                        │
│                                                                  │
│  Usuario ──► /auth/login (email, password)                      │
│                 │                                                │
│                 ▼                                                │
│  Auth Server:   ¿Email + password correctos?                     │
│                 │                                                │
│                 ▼                                                │
│                 Genera:                                          │
│                 • Access Token (JWT, expira en 15 min)           │
│                 • Refresh Token (opaco o JWT, expira en 7 días)  │
│                 │                                                │
│                 ▼                                                │
│  Usuario ◄────  Ambos tokens                                     │
│                                                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  2. USO NORMAL (CADA REQUEST)                                   │
│                                                                  │
│  Usuario ──► GET /api/orders                                    │
│             Header: Authorization: Bearer <access_token>        │
│                 │                                                │
│                 ▼                                                │
│  API Gateway o Middleware:                                       │
│                 1. Verificar firma JWT (¿el token es legítimo?)  │
│                 2. Verificar exp (¿no expiró?)                   │
│                 3. Verificar iss, aud (¿es para nosotros?)       │
│                 4. Extraer claims (sub, role, permissions)       │
│                 5. Pasar al servicio                              │
│                 │                                                │
│                 ▼                                                │
│  Servicio recibe: user_id="user_abc", role="CUSTOMER"           │
│  El servicio NO valida el JWT de nuevo (el gateway ya lo hizo). │
│                                                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  3. REFRESCO (CUANDO EL ACCESS TOKEN EXPIRA)                     │
│                                                                  │
│  Usuario ──► POST /auth/refresh                                 │
│             Body: { refresh_token: "rt_xyz..." }                │
│                 │                                                │
│                 ▼                                                │
│  Auth Server:   ¿El refresh token es válido y no revocado?      │
│                 │                                                │
│                 ▼                                                │
│                 Genera NUEVO Access Token (con claims actuales). │
│                 (Opcional: rota también el Refresh Token).      │
│                 │                                                │
│                 ▼                                                │
│  Usuario ◄────  Nuevo Access Token                               │
│                                                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  4. LOGOUT                                                       │
│                                                                  │
│  Usuario ──► POST /auth/logout                                  │
│             Body: { refresh_token: "rt_xyz..." }                │
│                 │                                                │
│                 ▼                                                │
│  Auth Server:   Marca el refresh token como REVOCADO.           │
│                 El access token actual sigue siendo válido      │
│                 hasta que expire (máx 15 minutos).               │
│                                                                  │
│  Nota: No hay forma de invalidar un JWT access token             │
│        inmediatamente SIN mantener un estado en el servidor      │
│        (lo cual derrota el propósito "stateless" de JWT).        │
│        Por eso los access tokens deben durar POCO.               │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Implementación de Validación de JWT (Nivel Producción)

```java
// ✅ ESTO es lo que debe hacer tu middleware de JWT en producción

public class JwtAuthenticator {
    private final JwkProvider jwkProvider; // Obtiene claves públicas del auth server
    
    public AuthenticatedUser authenticate(String authHeader) {
        // 1. ¿Hay token?
        if (authHeader == null || !authHeader.startsWith("Bearer ")) {
            throw new MissingTokenException();
        }
        String token = authHeader.substring(7);
        
        try {
            // 2. Decodificar sin verificar primero (para leer el kid)
            DecodedJWT decoded = JWT.decode(token);
            String keyId = decoded.getKeyId();
            
            // 3. Obtener la clave pública correcta (rotación de claves)
            Jwk jwk = jwkProvider.get(keyId);
            PublicKey publicKey = jwk.getPublicKey();
            
            // 4. Verificar firma + exp + iss + aud TODO JUNTO
            Algorithm algorithm = Algorithm.RSA256;
            JWTVerifier verifier = JWT.require(algorithm)
                .withIssuer("https://auth.miempresa.com")
                .withAudience("shopflow-api")
                .acceptLeeway(30)  // 30 segundos de tolerancia para clock skew
                .build();
            
            DecodedJWT verified = verifier.verify(token);
            
            // 5. Verificar claims adicionales
            if (verified.getSubject() == null) {
                throw new InvalidTokenException("Token sin subject");
            }
            
            // 6. ¿Está en la lista de revocación? (opcional, requiere Redis/BD)
            if (revocationList.isRevoked(verified.getId())) {
                throw new RevokedTokenException();
            }
            
            // 7. Construir el usuario autenticado
            return AuthenticatedUser.builder()
                .userId(verified.getSubject())
                .roles(verified.getClaim("roles").asList(String.class))
                .permissions(verified.getClaim("permissions").asList(String.class))
                .build();
                
        } catch (JWTVerificationException e) {
            // 8. Log detallado para debugging (pero sin el token completo)
            log.warn("JWT validation failed: {}", e.getMessage());
            throw new InvalidTokenException("Token inválido o expirado");
        }
    }
}
```

### Cuándo Usar JWT y Cuándo NO

```
✅ USA JWT cuando:
  • APIs stateless: no quieres consultar una BD en cada request.
  • Microservicios: pasar contexto de usuario entre servicios.
  • Federación: un sistema emite, otros verifican (con clave pública).
  • Mobile apps / SPAs: tokens que el cliente puede almacenar y enviar.

❌ NO uses JWT cuando:
  • Sesiones web tradicionales (server-side rendering):
    → Cookies de sesión tradicionales son más simples y seguras.
  • Necesitas revocación INMEDIATA (antes de la expiración del token):
    → JWT es inherentemente stateless. La revocación instantánea
      requiere estado (Redis/BD), lo que derrota el propósito.
  • Necesitas almacenar datos sensibles en el token:
    → El payload es Base64, no encriptado. Cualquiera puede leerlo.
      Usa JWE (JSON Web Encryption) si realmente necesitas payload secreto.
  • Tus tokens duran más de 1 hora:
    → Si el token dura 24h, un token robado = 24h de acceso.
      Mejor: access token 15min + refresh token 7d.
```

---

## 17.3 OAuth 2.0 — El Estándar que Debes Dominar

### OAuth 2.0 NO Es Autenticación

OAuth 2.0 es un framework de **autorización delegada**. Permite que un usuario conceda acceso limitado a sus recursos a una aplicación de terceros sin compartir sus credenciales.

**OAuth 2.0 te dice: "La aplicación X PUEDE acceder a tu Google Drive."**
**No te dice: "Tú ERES Juan Pérez."** (Para eso existe OpenID Connect.)

### Los 4 Roles de OAuth 2.0

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                  │
│  1. RESOURCE OWNER (Dueño del Recurso)                          │
│     El USUARIO. Es dueño de sus datos.                          │
│     Ejemplo: Juan, dueño de su Google Drive.                    │
│                                                                  │
│  2. CLIENT (Cliente)                                            │
│     La APLICACIÓN que quiere acceder a los datos.               │
│     Ejemplo: "PhotoPrinter", la app que imprime tus fotos.      │
│                                                                  │
│  3. AUTHORIZATION SERVER (Servidor de Autorización)             │
│     Emite tokens al Cliente con permiso del Dueño del Recurso.  │
│     Ejemplo: accounts.google.com                                │
│                                                                  │
│  4. RESOURCE SERVER (Servidor de Recursos)                      │
│     Tiene los datos protegidos. Acepta tokens del Auth Server.  │
│     Ejemplo: Google Drive API                                   │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Los Flujos (Grants) de OAuth 2.0 — Cuál Usar y Por Qué

#### Authorization Code + PKCE — EL FLUJO QUE DEBES USAR

Este es el flujo recomendado para aplicaciones web, móviles y SPAs. PKCE (Proof Key for Code Exchange, pronunciado "pixy") añade una capa de seguridad que previene ataques de interceptación del authorization code.

```
┌──────────────────────────────────────────────────────────────────┐
│                                                                   │
│ PASO 1: La app genera un CODE VERIFIER (secreto criptográfico)   │
│         y su CODE CHALLENGE (hash del verifier).                  │
│                                                                   │
│   const codeVerifier = generateRandomString(128);                 │
│   const codeChallenge = sha256(codeVerifier);                     │
│                                                                   │
│ PASO 2: La app redirige al usuario al Authorization Server       │
│         con el code_challenge.                                    │
│                                                                   │
│   GET https://auth.miempresa.com/authorize?                       │
│       response_type=code&                                         │
│       client_id=app_123&                                          │
│       redirect_uri=https://miapp.com/callback&                    │
│       code_challenge=ABC123&          ← hash del verifier         │
│       code_challenge_method=S256&     ← SHA-256                   │
│       scope=read:orders+write:orders&                             │
│       state=random_xyz123              ← anti-CSRF                │
│                                                                   │
│ PASO 3: El usuario inicia sesión y CONSiente.                     │
│         Auth Server redirige al redirect_uri con un CODE.         │
│                                                                   │
│   https://miapp.com/callback?code=AUTH_CODE_789&state=random_xyz123│
│                                                                   │
│ PASO 4: La app intercambia el CODE + CODE VERIFIER por TOKENS.   │
│         (El code solo se puede usar UNA VEZ)                     │
│                                                                   │
│   POST https://auth.miempresa.com/token                           │
│   grant_type=authorization_code                                   │
│   code=AUTH_CODE_789                                              │
│   redirect_uri=https://miapp.com/callback                         │
│   client_id=app_123                                               │
│   code_verifier=SECRET_VERIFIER_XYZ  ← EL SECRETO ORIGINAL        │
│                                                                   │
│   Auth Server verifica:                                           │
│     sha256(code_verifier) == code_challenge? OK                   │
│     → Emite access_token + refresh_token + id_token               │
│                                                                   │
│ ¿Por qué PKCE?                                                    │
│ Sin PKCE, un atacante que intercepta el CODE puede canjearlo     │
│ por tokens. Con PKCE, necesita también el code_verifier original. │
│ El atacante solo vio el code_challenge (hash), no el verifier.    │
│                                                                   │
└──────────────────────────────────────────────────────────────────┘
```

#### Client Credentials — Para Service-to-Service

```
Usa este flujo cuando NO hay un usuario humano. Es comunicación
máquina a máquina.

┌──────────┐     ┌────────────────┐     ┌──────────┐
│ Servicio │────►│  Auth Server   │────►│ Servicio │
│   A      │     │                │     │   B      │
│          │     │ POST /token    │     │          │
│          │     │ grant_type=    │     │          │
│          │     │ client_        │     │          │
│          │     │ credentials    │     │          │
│          │     │ client_id=     │     │          │
│          │     │ client_secret= │     │          │
│          │◄────│ access_token   │     │          │
│          │     │                │     │          │
│          │───► GET /api/data ──►│     │          │
│          │     Authorization:   │     │          │
│          │     Bearer <token>   │     │          │
└──────────┘     └────────────────┘     └──────────┘
```

```java
// Servicio A obtiene token para llamar a Servicio B
public class ServiceAClient {
    private String accessToken;
    private Instant tokenExpiry;
    
    private void refreshTokenIfNeeded() {
        if (accessToken == null || Instant.now().isAfter(tokenExpiry)) {
            TokenResponse response = authServer.requestToken(
                "grant_type=client_credentials",
                "client_id=" + CLIENT_ID,
                "client_secret=" + clientSecret,
                "scope=read:orders"
            );
            this.accessToken = response.getAccessToken();
            this.tokenExpiry = Instant.now().plusSeconds(response.getExpiresIn());
        }
    }
    
    public List<Order> getOrders() {
        refreshTokenIfNeeded();
        return httpClient.get("https://api.servicio-b.com/orders")
            .header("Authorization", "Bearer " + accessToken)
            .execute();
    }
}
```

#### Flujos que DEBES EVITAR

```
❌ IMPLICIT GRANT: OBSOLETO. No lo uses.
   El token se devolvía en el fragmento de URL (visible en el historial).
   Sin PKCE. Inseguro para SPAs modernas.
   
   → Reemplazo: Authorization Code + PKCE (funciona para SPAs también).

❌ RESOURCE OWNER PASSWORD CREDENTIALS: OBSOLETO.
   La app recibe directamente el username y password del usuario.
   Rompe el propósito de OAuth: "no compartas tu password con terceros".
   
   → Reemplazo: Authorization Code + PKCE.

❌ DEVICE CODE: Solo para dispositivos sin navegador (TVs, IoT, impresoras).
   No lo uses para web/mobile apps normales.
```

---

## 17.4 OpenID Connect (OIDC) — Autenticación Sobre OAuth 2.0

OAuth 2.0 te dice qué puede hacer una app. OpenID Connect te dice QUIÉN es el usuario.

```
OAuth 2.0:  "Esta app PUEDE leer tus fotos de Google Photos."
            → Access Token

OIDC:       "Esta app SABE que eres juan@gmail.com y tu nombre es Juan Pérez."
            → ID Token (además del Access Token)
```

### El ID Token — Lo Que OIDC Añade

```json
// ID Token (siempre JWT)
{
  "iss": "https://accounts.google.com",       // Quién autenticó
  "sub": "1234567890",                         // ID del usuario (estable)
  "aud": "app_123",                            // Para quién es este token
  "exp": 1705392000,
  "iat": 1705305600,
  
  // ─── Claims del usuario (lo que OIDC añade sobre OAuth) ───
  "email": "juan.perez@gmail.com",
  "email_verified": true,
  "name": "Juan Pérez",
  "given_name": "Juan",
  "family_name": "Pérez",
  "picture": "https://lh3.googleusercontent.com/.../photo.jpg",
  "locale": "es-MX",
  
  // ─── Metadata de autenticación ───
  "auth_time": 1705305600,                     // Cuándo se autenticó
  "amr": ["password", "mfa"],                  // Authentication Methods Reference
  "acr": "urn:mace:incommon:iap:silver"        // Authentication Context Class Reference
}
```

### Cuándo Usas Cada Token

```
┌─────────────────────────────────────────────────────────────┐
│                                                              │
│  ACCESS TOKEN: "Puedo actuar en nombre del usuario"          │
│  • Lo envías a las APIs (Bearer token).                     │
│  • Tiene scopes: read:orders, write:profile.                │
│  • Vida corta (15-60 min).                                   │
│  • NO contiene información del usuario (solo sub y scopes). │
│  • Es opaco o JWT. El cliente no debe inspeccionarlo.       │
│                                                              │
│  ID TOKEN: "Sé quién es el usuario"                         │
│  • SOLO lo usa el CLIENTE (tu app) para saber quién es.    │
│  • NUNCA lo envías a tus APIs como autenticación.           │
│  • Contiene claims del usuario: nombre, email, foto.        │
│  • Es SIEMPRE JWT. El cliente lo decodifica y verifica.     │
│  • Vida corta (15-60 min).                                   │
│                                                              │
│  REFRESH TOKEN: "Puedo obtener nuevos access tokens"         │
│  • SOLO se envía al Authorization Server.                   │
│  • NUNCA a las APIs de recursos.                            │
│  • Vida larga (días-semanas). Puede ser revocado.           │
│  • Almacenamiento seguro: HttpOnly cookie o secure storage. │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## 17.5 Modelos de Autorización — Más Allá de "Es Admin o No"

### RBAC — Role-Based Access Control

El más común y el más mal implementado.

```java
// ❌ RBAC MAL HECHO: Checkea strings en cada método
public void deleteOrder(String orderId, User user) {
    if (user.getRole().equals("ADMIN") || user.getRole().equals("MANAGER")) {
        orderRepo.delete(orderId);
    } else {
        throw new ForbiddenException();
    }
}
// 50 endpoints. 50 ifs. Si añades el rol "SUPERVISOR", tocas 50 archivos.

// ✅ RBAC BIEN HECHO: Centralizado con permisos, no roles en código
@PreAuthorize("hasPermission('orders:delete')")
public void deleteOrder(String orderId) {
    orderRepo.delete(orderId);
}

// Configuración centralizada de roles → permisos (fuera del código)
// admin → [orders:delete, orders:create, users:manage, ...]
// manager → [orders:delete, orders:create]
// supervisor → [orders:read_all]
// user → [orders:read_own]
```

### ABAC — Attribute-Based Access Control

Decisiones basadas en atributos del usuario, recurso y contexto.

```
Política: "Un MANAGER puede aprobar descuentos de hasta 30%
           en pedidos de su PROPIO departamento,
           solo en HORARIO LABORAL."

Atributos del SUJETO (quién):
  • Role: MANAGER
  • Department: "Electrónicos"

Atributos del RECURSO (qué):
  • Tipo: "Descuento"
  • Valor: 25%
  • Department: "Electrónicos"

Atributos del CONTEXTO (cuándo/dónde):
  • Hora actual: 14:30 (horario laboral ✓)
  • IP de origen: 10.0.1.50 (red corporativa ✓)

Decisión:
  ✓ Role OK + Department coincide + Descuento ≤30% + Horario laboral → PERMITIDO
  ✗ Si alguno falla → DENEGADO
```

### OPA (Open Policy Agent) — Autorización como Código

```rego
# policy.rego — El archivo de políticas de autorización

package shopflow.authz

# ─── Regla por defecto: DENEGAR ───
default allow = false

# ─── Admins pueden todo ───
allow {
    input.user.role == "ADMIN"
}

# ─── Usuarios pueden leer SUS propios pedidos ───
allow {
    input.action == "orders:read"
    input.user.id == input.resource.owner_id
}

# ─── Managers pueden leer TODOS los pedidos ───
allow {
    input.action == "orders:read"
    input.user.role == "MANAGER"
}

# ─── Eliminar pedidos: solo el owner o admin ───
allow {
    input.action == "orders:delete"
    input.user.role == "ADMIN"
}
allow {
    input.action == "orders:delete"
    input.user.id == input.resource.owner_id
    input.resource.status == "PENDING"  # Solo pedidos pendientes
}

# ─── Aprobar descuentos: solo managers en su departamento ───
allow {
    input.action == "discounts:approve"
    input.user.role == "MANAGER"
    input.user.department == input.resource.department
    input.resource.discount_percent <= 30
    is_business_hours(input.context.time)
}

# ─── Función auxiliar ───
is_business_hours(time) {
    hour := time_utils.get_hour(time)
    hour >= 9
    hour < 18
}
```

### Cómo se Integra OPA en tu Arquitectura

```
┌──────────┐     ┌──────────┐     ┌──────────┐     ┌──────────┐
│ Servicio │────►│   OPA    │────►│ Servicio │     │          │
│  llama   │     │ (sidecar │     │ procesa  │     │          │
│  al API  │     │  o API)  │     │          │     │          │
└──────────┘     └──────────┘     └──────────┘     └──────────┘
                      │
                      │ Las políticas están en un repo Git
                      │ Se actualizan sin redeploy de servicios
                      │
               ┌──────▼──────┐
               │  Git Repo   │
               │  policies/  │
               │  authz.rego │
               └─────────────┘
```

---

## 17.6 SSO (Single Sign-On) — Una Identidad, Múltiples Aplicaciones

### Cómo Funciona Realmente

```
┌─────────────────────────────────────────────────────────────┐
│                                                              │
│  1. Usuario intenta acceder a App A                          │
│     → App A redirige a Identity Provider (IdP)               │
│                                                              │
│  2. IdP verifica: ¿Ya tienes sesión?                         │
│     ├── SÍ → Redirige de vuelta con token (sin pedir login)  │
│     └── NO  → Muestra pantalla de login                      │
│               → Usuario se autentica UNA VEZ                 │
│               → IdP crea sesión (cookie en dominio del IdP)  │
│                                                              │
│  3. Usuario intenta acceder a App B                          │
│     → App B redirige al MISMO IdP                            │
│     → IdP ve la sesión YA EXISTENTE (paso 2)                 │
│     → NO pide login de nuevo                                 │
│     → Redirige de vuelta con token para App B                │
│                                                              │
│  4. Usuario cierra sesión en App A                           │
│     → App A redirige a IdP para logout                       │
│     → IdP destruye la sesión CENTRAL                         │
│     → TODAS las apps redirigidas al IdP pedirán login        │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Protocolos de SSO

```
SAML 2.0:   El estándar enterprise. XML pesado pero maduro.
            Salesforce, Office 365, Workday lo usan.
            
OIDC:       El estándar moderno. Más simple, basado en OAuth 2.0.
            Google, Apple, Microsoft, Auth0, Okta lo soportan.
            
WS-Fed:     Legado Microsoft. En desuso.

Para sistemas NUEVOS: OIDC. Para integrar con enterprise legacy: SAML.
```

---

## 17.7 Autenticación Multi-Factor (MFA)

### Los Tres Factores de Autenticación

```
┌──────────────────────────────────────────────────────────────┐
│                                                               │
│  Factor 1: ALGO QUE SABES (Knowledge)                        │
│  Password, PIN, respuesta secreta                             │
│  Problema: Se puede adivinar, phishing, keylogger.           │
│                                                               │
│  Factor 2: ALGO QUE TIENES (Possession)                      │
│  Teléfono (SMS/App authenticator), YubiKey, tarjeta NFC      │
│  Problema: Se puede perder, robar, SIM swapping.             │
│                                                               │
│  Factor 3: ALGO QUE ERES (Inherence)                         │
│  Huella dactilar, Face ID, iris, voz                          │
│  Problema: No se puede cambiar si es comprometido.            │
│           Falsos positivos/negativos.                         │
│                                                               │
│  MFA = Usar 2 o más factores DIFERENTES.                     │
│  Password + SMS = ✓ (Knowledge + Possession)                 │
│  Password + PIN  = ✗ (Dos del mismo factor: Knowledge)       │
│                                                               │
└──────────────────────────────────────────────────────────────┘
```

### TOTP — El Método Más Común (Google Authenticator)

```
TOTP (Time-based One-Time Password):

El servidor y el cliente comparten un SECRET (generado al configurar MFA).

Cada 30 segundos:
  código = HMAC-SHA1(secret, timestamp_actual / 30)
  mostrar los últimos 6 dígitos del hash

El servidor calcula lo mismo. Si coinciden → MFA exitoso.

No requiere internet en el dispositivo.
No requiere SMS (costoso, vulnerable a SIM swapping).
```

---

## 17.8 Almacenamiento de Passwords — Lo Que TODO Sistema Debe Hacer

```java
// ❌ LO QUE NO DEBES HACER — y miles de sistemas lo hacen

// Texto plano
user.setPassword("mipassword123");
// Si la BD se filtra, el atacante ve TODAS las contraseñas.

// Hash simple (MD5, SHA-1, SHA-256)
user.setPassword(md5("mipassword123")); // "482c811da5d5b4bc6d497ffa98491e38"
// Ataque de rainbow table: tablas precalculadas de hash → password.

// Hash con salt fijo
user.setPassword(sha256("mipassword123" + "MI_SALT_FIJO"));
// Si el salt está en el código (o es igual para todos), 
// el atacante recalcula la tabla para ESE salt.


// ✅ LO QUE SÍ DEBES HACER

// bcrypt con salt aleatorio y factor de costo
String password = "mipassword123";
String hashedPassword = BCrypt.hashpw(password, BCrypt.gensalt(12));
// $2a$12$LJ3m4ys3GZ0YKh8NqF5N5u8j2X5k3M7pQ9rS1tU4vW6xY7zA0bC1

// bcrypt:
//   • Genera un salt aleatorio ÚNICO para cada password.
//   • Incluye el salt en el hash resultante (no necesitas guardarlo aparte).
//   • El factor de costo (12) determina cuántas iteraciones.
//     2^12 = 4096 iteraciones.
//     Cada incremento de 1 duplica el tiempo.
//     En 2024: factor 12-14 es estándar.
//   • Diseñado para ser LENTO (a propósito). Un ataque de fuerza bruta
//     que prueba 1,000 passwords/segundo con MD5 prueba 5/segundo con bcrypt.

// Verificar password (bcrypt extrae el salt del hash almacenado)
boolean valid = BCrypt.checkpw(password, hashedPassword);
```

```python
# Python: usa bcrypt o argon2
import bcrypt

# Hash
hashed = bcrypt.hashpw(password.encode(), bcrypt.gensalt(rounds=12))

# Verify
if bcrypt.checkpw(password.encode(), hashed):
    print("Password correcto")
```

---

## 17.9 Checklist de Auth del Arquitecto

```
ANTES de lanzar tu sistema, verifica:

AUTENTICACIÓN:
[ ] ¿Passwords se almacenan con bcrypt/argon2? (NUNCA texto plano/SHA/MD5)
[ ] ¿El factor de costo de bcrypt es ≥12?
[ ] ¿MFA está disponible (al menos para admins y acciones críticas)?
[ ] ¿Hay rate limiting en el endpoint de login? (máx 5 intentos/min por IP/usuario)
[ ] ¿Hay lockout temporal tras N intentos fallidos?
[ ] ¿Los tokens de sesión tienen expiración razonable?
[ ] ¿Logout invalida realmente la sesión (no solo borra la cookie del cliente)?
[ ] ¿Las credenciales por defecto se fuerzan a cambiar en el primer login?

AUTORIZACIÓN:
[ ] ¿Cada endpoint verifica autorización (no solo autenticación)?
[ ] ¿La lógica de autorización está CENTRALIZADA (no if/else dispersos)?
[ ] ¿El esquema por defecto es DENEGAR (allowlist, no denylist)?
[ ] ¿Los IDs de recursos se validan contra el usuario que los pide?
    (Usuario 456 pide pedido 789 → ¿El pedido 789 es del usuario 456?)
[ ] ¿Hay logs de auditoría para acciones críticas (cambios de rol, eliminaciones)?

OAUTH / OIDC / SSO:
[ ] ¿Usas Authorization Code + PKCE (no Implicit)?
[ ] ¿Los redirect_uri están en lista blanca EXACTA (no wildcards)?
[ ] ¿El parámetro state se usa para prevenir CSRF?
[ ] ¿Los refresh tokens se pueden revocar?
[ ] ¿Los ID tokens se validan completamente (iss, aud, exp, nonce)?

JWT:
[ ] ¿Los access tokens expiran en ≤30 minutos?
[ ] ¿Verificas iss, aud, exp, firma en cada token?
[ ] ¿Soportas rotación de claves (JWKS endpoint)?
[ ] ¿NUNCA pones datos sensibles en el payload?
```

---

> **Reflexión del capítulo**: La autenticación y autorización son el candado de tu castillo digital. Puedes tener los muros más gruesos, los fosos más profundos, los dragones más feroces... pero si la cerradura es débil, nada de eso importa. Un error de auth no es un bug. Es una brecha. Y las brechas salen en las noticias. Invierte el tiempo en entender auth a profundidad. Tus usuarios confían en ti sus datos. No los traiciones con un "if (user.role == 'ADMIN')" en 50 archivos.

---

← [Capítulo anterior](16-seguridad.md) | [Inicio](README.md) | [Capítulo siguiente →](18-escalabilidad.md)
