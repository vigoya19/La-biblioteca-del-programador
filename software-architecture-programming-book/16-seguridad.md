# Capítulo 16: Seguridad en Arquitectura de Software — El Mini-Libro Definitivo

> "Hay dos tipos de empresas: las que han sido hackeadas y las que no saben que han sido hackeadas." — Dmitri Alperovitch

## 16.1 Los Tres Pilares que Sostienen Todo (CIA Triad)

Antes de hablar de firewalls, tokens o encriptación, necesitas entender qué estás protegiendo. La tríada CIA es el modelo mental más fundamental de la seguridad informática. Todo lo demás son implementaciones de estos tres conceptos.

```
┌─────────────────────────────────────────────────────────────┐
│                                                              │
│                  CONFIDENCIALIDAD                            │
│                  (Confidentiality)                           │
│                  Solo los autorizados pueden LEER los datos. │
│                                                              │
│  Ejemplo: Solo tú y tu médico pueden ver tu historial       │
│           clínico. El recepcionista no. El farmaceútico no. │
│                                                              │
│  Si falla: Filtración de datos.                              │
│  Mitigación: Encriptación, control de acceso, clasificación.│
│                                                              │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│                      INTEGRIDAD                              │
│                      (Integrity)                             │
│           Los datos no se alteran sin autorización.          │
│                                                              │
│  Ejemplo: Tu saldo bancario no puede cambiar de $1,000      │
│           a $10 sin que el banco lo registre.                │
│                                                              │
│  Si falla: Corrupción de datos, fraude.                      │
│  Mitigación: Firmas digitales, hash, checksums, audit logs. │
│                                                              │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│                    DISPONIBILIDAD                             │
│                    (Availability)                            │
│       El sistema está accesible cuando se necesita.          │
│                                                              │
│  Ejemplo: El sistema de emergencias 911 debe funcionar      │
│           SIEMPRE.                                            │
│                                                              │
│  Si falla: Denegación de servicio, pérdida de negocio.       │
│  Mitigación: Redundancia, DDoS protection, backups.         │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Cómo la Tríada CIA Guía Tus Decisiones

Cada decisión de seguridad es un trade-off entre estos tres pilares.

```
Escenario: Decides encriptar TODOS los datos en reposo.

Confidencialidad: SUBE ✓  (los datos están protegidos en disco)
Integridad:      IGUAL  (la encriptación no evita modificaciones)
Disponibilidad:  BAJA ✗   (encriptar/desencriptar añade latencia,
                          perder la clave de encriptación = perder
                          los datos para siempre)

Decisión informada: Encriptar datos sensibles (PII, tarjetas).
                    No encriptar assets públicos (imágenes de producto).
                    
Escenario: Añades autenticación de dos factores (MFA).

Confidencialidad: SUBE ✓✓ (mucho más difícil de vulnerar)
Integridad:      SUBE ✓  (el atacante no puede hacerse pasar por ti)
Disponibilidad:  BAJA ✗   (si pierdes tu segundo factor, no puedes
                          acceder. Necesitas proceso de recovery.)
```

---

## 16.2 Defensa en Profundidad — El Castillo Medieval

> "La seguridad no es un producto que compras. Es una arquitectura que diseñas."

Una sola barrera de seguridad es un punto único de fallo. La defensa en profundidad asume que CADA capa fallará eventualmente, y pone otra capa detrás.

### Las 7 Capas de Defensa que Debes Conocer

```
CAPA 1: Perímetro de Red
┌─────────────────────────────────────────────────────────────┐
│ WAF (Web Application Firewall), DDoS Protection, Firewall   │
│                                                              │
│ ¿Qué protege? Ataques volumétricos, SQL injection básico,   │
│               cross-site scripting desde fuera.              │
│                                                              │
│ Si falla:   El atacante llega a tu red interna.             │
│ Entonces:   La Capa 2 lo detiene.                           │
└─────────────────────────────────────────────────────────────┘
                          │
                          ▼
CAPA 2: Gateway de API
┌─────────────────────────────────────────────────────────────┐
│ API Gateway + Rate Limiting + Input Validation              │
│                                                              │
│ ¿Qué protege? Abuso de API, fuerza bruta, payloads malicios.│
│                                                              │
│ Si falla:   El atacante puede llamar a tus servicios.       │
│ Entonces:   La Capa 3 lo detiene.                           │
└─────────────────────────────────────────────────────────────┘
                          │
                          ▼
CAPA 3: Autenticación
┌─────────────────────────────────────────────────────────────┐
│ "¿Quién eres?" — Credenciales, MFA, biometría, certificados│
│                                                              │
│ Si falla:   El atacante se hace pasar por un usuario válido.│
│ Entonces:   La Capa 4 lo detiene.                           │
└─────────────────────────────────────────────────────────────┘
                          │
                          ▼
CAPA 4: Autorización
┌─────────────────────────────────────────────────────────────┐
│ "¿Puedes hacer esto?" — RBAC, ABAC, políticas OPA          │
│                                                              │
│ Si falla:   Un usuario autenticado accede a datos ajenos.   │
│ Entonces:   La Capa 5 lo registra y la 6 lo audita.         │
└─────────────────────────────────────────────────────────────┘
                          │
                          ▼
CAPA 5: Cifrado en Tránsito y Reposo
┌─────────────────────────────────────────────────────────────┐
│ TLS 1.3, mTLS, AES-256 en disco, KMS para claves           │
│                                                              │
│ Si falla:   Un atacante que intercepta tráfico o roba discos│
│             puede leer los datos.                            │
└─────────────────────────────────────────────────────────────┘
                          │
                          ▼
CAPA 6: Auditoría y Registro
┌─────────────────────────────────────────────────────────────┐
│ Logs de acceso, cambios de configuración, eventos anómalos  │
│                                                              │
│ Si falla:   No sabes que fuiste atacado. No puedes investigar│
│             cómo pasó, qué se llevaron, quién fue.           │
└─────────────────────────────────────────────────────────────┘
                          │
                          ▼
CAPA 7: Respuesta y Recuperación
┌─────────────────────────────────────────────────────────────┐
│ Plan de respuesta a incidentes, backups inmutables,         │
│ disaster recovery, comunicación de brechas.                 │
│                                                              │
│ Cuando todo lo demás falló, esto minimiza el daño.          │
└─────────────────────────────────────────────────────────────┘
```

### Ejemplo Concreto: Cómo un Ataque Atraviesa las Capas

```
Escenario: Un atacante intenta robar datos de tarjetas de crédito.

Intento 1: Escaneo de puertos desde internet.
  → CAPA 1 lo bloquea (Security Group solo permite 443).

Intento 2: SQL Injection en el endpoint de búsqueda.
  → CAPA 1 lo detecta (WAF con reglas OWASP).
  → CAPA 2 también: API Gateway valida inputs, rechaza patrones SQL.

Intento 3: El atacante obtiene credenciales de un empleado (phishing).
  → CAPA 1-2: pasa (tráfico legítimo).
  → CAPA 3: ¡MFA! El atacante no tiene el segundo factor. Bloqueado.

Intento 4: El atacante roba el laptop de un empleado con sesión abierta.
  → CAPA 1-2: pasa (dispositivo legítimo).
  → CAPA 3: pasa (sesión activa).
  → CAPA 4: El empleado es "agente de soporte", solo puede ver
           datos de cliente, no tarjetas completas. Bloqueado.

Intento 5: El atacante compromete al administrador del sistema.
  → CAPA 1-2-3-4: pasa (admin tiene acceso legítimo).
  → CAPA 5: Los datos de tarjeta están encriptados con KMS.
           El atacante ve texto cifrado. Necesita acceso al KMS.
  → CAPA 6: Cada acceso del admin queda registrado.
           El SOC detecta acceso a KMS a las 3 AM (anómalo).
           Alerta disparada. Investigación iniciada.
  → CAPA 7: Se revocan credenciales del admin. Se inicia
           protocolo de respuesta a incidentes.

Resultado: El atacante atravesó 4 capas. Pero fue detenido
           en la capa 5 (datos encriptados) y detectado en la 6.
```

---

## 16.3 OWASP Top 10 — Lo Que Todo Arquitecto Debe Saber

OWASP (Open Web Application Security Project) publica cada pocos años el Top 10 de vulnerabilidades más críticas en aplicaciones web. No necesitas ser experto en cada una, pero sí saber qué son y cómo tu arquitectura las mitiga.

```
OWASP Top 10 2021 — Tu checklist de arquitecto:

┌─────────────────────────────────────────────────────────────────────┐
│                                                                      │
│ #1 BROKEN ACCESS CONTROL (Control de Acceso Roto)                   │
│                                                                      │
│ ¿Qué es?   Un usuario puede acceder a datos o funciones que no      │
│            debería. El ataque #1 en aplicaciones reales.            │
│                                                                      │
│ Ejemplo real: Cambiar el ID en la URL.                              │
│   GET /api/orders/12345 → ves tu pedido.                            │
│   GET /api/orders/12346 → ves el pedido de otro. ¡Sin authZ!        │
│                                                                      │
│ Mitigación arquitectónica:                                           │
│   • Autorización en cada endpoint. No asumas "nadie encontrará esto."│
│   • Deny by default: si no está explícitamente permitido, denegar.   │
│   • Validar ownership: ¿el usuario 456 es dueño del pedido 123?      │
│   • Usar UUIDs, no IDs secuenciales (reduce enumeración).            │
│   • Centralizar autorización (gateway, middleware, no if/else).      │
│                                                                      │
│ Cómo detectarlo:                                                     │
│   Para cada endpoint pregúntate:                                     │
│   "¿Un usuario autenticado normal puede acceder a datos de otro?"    │
│   Si la respuesta no es un rotundo NO, tienes un problema.           │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│                                                                      │
│ #2 CRYPTOGRAPHIC FAILURES (Fallos Criptográficos)                   │
│                                                                      │
│ ¿Qué es?   Usar criptografía débil, incorrecta o ausente.           │
│                                                                      │
│ Ejemplos:                                                            │
│   • MD5 o SHA-1 para passwords (usa bcrypt/argon2).                 │
│   • HTTP en vez de HTTPS en producción.                              │
│   • AES con ECB mode (inseguro, revela patrones).                   │
│   • Hardcodear claves de encriptación en el código.                  │
│   • TLS 1.0 (obsoleto, vulnerable a POODLE, BEAST).                 │
│   • "Nuestra propia implementación de encriptación".                 │
│                                                                      │
│ Reglas de oro:                                                       │
│   • NUNCA inventes tu propia criptografía. Usa bibliotecas estándar. │
│   • TLS 1.3 como mínimo (1.2 aceptable si no puedes 1.3).           │
│   • Passwords: bcrypt o argon2id (NUNCA MD5, SHA-1, SHA-256).       │
│   • Datos en reposo: AES-256-GCM.                                    │
│   • Claves: en KMS/HSM, nunca en código ni archivos de configuración.│
│   • Rota claves periódicamente.                                      │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│                                                                      │
│ #3 INJECTION (Inyección)                                            │
│                                                                      │
│ ¿Qué es?   Datos no confiables se interpretan como código.          │
│                                                                      │
│ Tipos:   SQL injection, LDAP injection, OS command injection,       │
│          NoSQL injection, XSS (la inyección en el navegador).       │
│                                                                      │
│ Ejemplo SQLi:                                                        │
│   Input del usuario: ' OR '1'='1                                    │
│   Query resultante: SELECT * FROM users WHERE name = '' OR '1'='1'  │
│   → Devuelve TODOS los usuarios.                                     │
│                                                                      │
│ Mitigación arquitectónica:                                           │
│   • Prepared statements / parameterized queries SIEMPRE.             │
│   • ORMs con protección integrada (Hibernate, ActiveRecord).         │
│   • Validación de input: whitelist, no blacklist.                    │
│   • Principio de mínimo privilegio en BD (app user no tiene DROP).   │
│   • WAF como segunda barrera (no como única).                        │
│                                                                      │
│ Para NoSQL:                                                          │
│   • Validar tipos (si esperas string, rechaza objetos JSON).         │
│   • Usar librerías de sanitización específicas.                      │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│                                                                      │
│ #4 INSECURE DESIGN (Diseño Inseguro)                                │
│                                                                      │
│ ¿Qué es?   La arquitectura misma es insegura. Faltan controles      │
│            desde el diseño, no por errores de implementación.        │
│                                                                      │
│ Ejemplos:                                                            │
│   • No hay límite en upload de archivos → DoS llenando disco.       │
│   • No hay rate limiting → fuerza bruta de passwords.               │
│   • IDs de usuario secuenciales → enumeración de usuarios.           │
│   • Sin logs de auditoría → ataque no detectado por meses.           │
│   • "Confiamos en que el frontend valida" → backend sin validación.  │
│                                                                      │
│ Mitigación:                                                          │
│   • Threat modeling desde el diseño (STRIDE).                        │
│   • Security user stories en el backlog.                             │
│   • "¿Qué podría salir mal?" en cada sesión de diseño.               │
│   • Principio de mínimo privilegio en cada componente.               │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│                                                                      │
│ #5 SECURITY MISCONFIGURATION (Mala Configuración de Seguridad)      │
│                                                                      │
│ ¿Qué es?   Componentes con configuración insegura por defecto.      │
│                                                                      │
│ Ejemplos reales:                                                     │
│   • S3 bucket público ("porque era más fácil para testing").         │
│   • MongoDB sin password en internet (pasó: 150M+ registros).        │
│   • Debug mode activado en producción (revela stack traces).         │
│   • CORS configurado como Access-Control-Allow-Origin: *.            │
│   • Directorio listing activado en nginx.                            │
│   • Kubernetes dashboard expuesto sin auth.                          │
│                                                                      │
│ Mitigación:                                                          │
│   • Infrastructure as Code (Terraform, CloudFormation).              │
│     No configures nada manualmente.                                  │
│   • Hardening guides para cada componente (CIS Benchmarks).          │
│   • Escaneo automático de configuraciones (Prowler, ScoutSuite).     │
│   • "Secure by default": que la configuración insegura sea la difícil.│
└─────────────────────────────────────────────────────────────────────┘

#6-#10 (Vulnerable Components, Auth Failures, Integrity Failures,
        Logging Failures, SSRF) — igualmente importantes, ver OWASP completo.
```

---

## 16.4 PCI-DSS — Lo Que Necesitas Saber Si Manejas Pagos

### ¿Qué Es PCI-DSS?

PCI-DSS (Payment Card Industry Data Security Standard) es el estándar de seguridad para cualquier organización que almacene, procese o transmita datos de tarjetas de crédito. No es una ley, es un contrato con las marcas de tarjetas (Visa, Mastercard, etc.). Pero violarlo puede significar multas de $5,000 a $100,000 por MES y la prohibición de aceptar tarjetas.

**En español simple: si manejas pagos con tarjeta, PCI-DSS no es opcional. Es el costo de hacer negocios.**

### Los 12 Requisitos de PCI-DSS (Versión Amigable para Arquitectos)

```
┌──────────────────────────────────────────────────────────────────┐
│ REQUISITOS DE RED Y SEGURIDAD PERIMETRAL                         │
│                                                                   │
│ 1. Firewall: Proteger los datos de tarjeta con firewalls.        │
│    → Security Groups + WAF en AWS. Segmentación de red (VPC).    │
│                                                                   │
│ 2. Configuraciones seguras: No usar defaults de fábrica.         │
│    → Cambiar contraseñas default de TODO (routers, APs, servers).│
│    → Hardening de imágenes Docker, AMIs.                          │
└──────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────┐
│ REQUISITOS DE PROTECCIÓN DE DATOS                                │
│                                                                   │
│ 3. Proteger datos almacenados: Encriptación + control de acceso. │
│    → AES-256 para datos de tarjeta en reposo.                     │
│    → NUNCA almacenar CVV (los 3 dígitos de atrás) después de     │
│      autorizar. Es ilegal bajo PCI-DSS.                           │
│    → Enmascarar PAN (Primary Account Number): mostrar solo       │
│      últimos 4 dígitos (**** **** **** 1234).                     │
│                                                                   │
│ 4. Encriptar en tránsito: TLS para redes abiertas/públicas.      │
│    → HTTPS en todos los endpoints que tocan datos de tarjeta.     │
│    → TLS 1.2+ (1.3 recomendado).                                  │
└──────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────┐
│ REQUISITOS DE GESTIÓN DE VULNERABILIDADES                        │
│                                                                   │
│ 5. Antivirus/Antimalware: Proteger sistemas contra malware.      │
│    → En cloud: usar imágenes base escaneadas.                     │
│    → ClamAV o similar en entornos Linux.                          │
│                                                                   │
│ 6. Sistemas seguros: Parchear y actualizar regularmente.         │
│    → Automated patching (AWS Systems Manager, K8s updates).       │
│    → Vulnerability scanning trimestral (Nessus, Qualys).          │
└──────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────┐
│ REQUISITOS DE CONTROL DE ACCESO                                  │
│                                                                   │
│ 7. Need-to-know: Acceso mínimo necesario a datos de tarjeta.     │
│    → "¿Este empleado necesita ver el número completo de tarjeta?" │
│      La respuesta casi siempre es NO.                             │
│    → Segmentación de datos: el servicio de analytics NO ve PAN.   │
│                                                                   │
│ 8. Identificación única: Cada usuario con ID único.              │
│    → No compartir cuentas. Cada acción atribuible.                 │
│                                                                   │
│ 9. Acceso físico: Restringir acceso físico a sistemas con datos. │
│    → En cloud: IAM roles, no accesos SSH directos a servidores.  │
└──────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────┐
│ REQUISITOS DE MONITOREO Y PRUEBAS                                 │
│                                                                   │
│ 10. Monitoreo: Registrar y monitorear todo acceso a datos.       │
│     → CloudTrail, VPC Flow Logs, application audit logs.         │
│     → Alertas en tiempo real para accesos anómalos.               │
│                                                                   │
│ 11. Pruebas de seguridad: Testear sistemas y procesos.            │
│     → Penetration testing anual por firma aprobada (ASV).         │
│     → Escaneo trimestral de vulnerabilidades externas.            │
│     → SAST + DAST en CI/CD para cada release.                     │
│                                                                   │
│ 12. Política de seguridad: Documentar y mantener políticas.       │
│     → Política de seguridad de la información documentada.        │
│     → Plan de respuesta a incidentes.                             │
│     → Evaluación de riesgos anual.                                │
└──────────────────────────────────────────────────────────────────┘
```

### La Estrategia que Ahorra Millones: Tokenización + Iframe

El error más caro que cometen los arquitectos novatos es diseñar sistemas donde los datos de tarjeta pasan por sus servidores. Cuando eso pasa, **todo tu sistema entra en scope de PCI-DSS**. Cada microservicio, cada base de datos, cada log, cada bucket S3 debe cumplir PCI-DSS. El costo de compliance se multiplica por 20.

**La solución elegante que usa la industria:**

```
┌────────────────────────────────────────────────────────────────┐
│                                                                │
│  Usuario ──► Tu Frontend ──► Iframe de Stripe/MercadoPago     │
│  (app web)    (tu código)     (código de Stripe, NO tuyo)     │
│                                    │                           │
│                                    ▼                           │
│                        Stripe recibe los datos de tarjeta       │
│                        DIRECTAMENTE del navegador del usuario. │
│                        Tus servidores NUNCA ven la tarjeta.    │
│                                    │                           │
│                                    ▼                           │
│                        Stripe te devuelve un TOKEN             │
│                        (ej: tok_1A2b3C4d5E).                   │
│                                    │                           │
│                                    ▼                           │
│                        Tu backend recibe el TOKEN.             │
│                        Envía el TOKEN a Stripe para cobrar.    │
│                                    │                           │
│                                    ▼                           │
│                        Stripe procesa el pago.                 │
│                        Tu backend guarda SOLO el payment_id.   │
│                        NUNCA guardas datos de tarjeta.          │
│                                                                │
│  Resultado: Tu scope PCI-DSS se reduce DRÁSTICAMENTE.         │
│  Pasas de cumplir 300+ controles a cumplir ~30 (SAQ-A).       │
│  Diferencia de costo de compliance: ~$10,000 vs ~$200,000/año.│
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

```javascript
// Frontend: Usa Stripe Elements (iframe de Stripe)
// Los datos de tarjeta NUNCA tocan tu servidor

const stripe = Stripe('pk_live_tu_public_key'); // Public key, seguro en frontend

// Crear el elemento de tarjeta (esto es un iframe alojado por Stripe)
const cardElement = elements.create('card');
cardElement.mount('#card-element'); // Se renderiza en tu página pero es DE STRIPE

// Cuando el usuario hace submit:
const { paymentMethod, error } = await stripe.createPaymentMethod({
  type: 'card',
  card: cardElement,
});

// Envías el ID del PaymentMethod a TU backend (NO datos de tarjeta)
await fetch('/api/payments', {
  method: 'POST',
  body: JSON.stringify({
    paymentMethodId: paymentMethod.id,  // pm_1A2b3C...
    orderId: 'order-123',
    amount: 9999,  // $99.99 en centavos
  }),
});
```

```java
// Backend: Recibes el PaymentMethod ID, NUNCA la tarjeta

@PostMapping("/api/payments")
public PaymentResponse processPayment(@RequestBody PaymentRequest request) {
    // request.getPaymentMethodId() = "pm_1A2b3C..."
    // NUNCA ves el número de tarjeta. Stripe lo maneja.
    
    PaymentIntent intent = PaymentIntent.create(
        PaymentIntentCreateParams.builder()
            .setAmount(request.getAmount())
            .setCurrency("usd")
            .setPaymentMethod(request.getPaymentMethodId())
            .setConfirm(true)  // Cobra inmediatamente
            .build()
    );
    
    // Guardas SOLO el PaymentIntent ID en tu BD
    paymentRepository.save(new Payment(
        intent.getId(),        // "pi_9Z8y..."
        request.getOrderId(),
        intent.getStatus()     // "succeeded"
    ));
    
    return new PaymentResponse(intent.getId(), intent.getStatus());
}
```

---

## 16.5 STRIDE — Cómo Pensar como un Atacante (Para Defenderte)

STRIDE es una metodología de Microsoft para identificar amenazas durante el diseño. Cada letra es una categoría de ataque. Para cada componente de tu sistema, te preguntas: ¿puede sufrir este tipo de ataque?

```
┌────────────────────────────────────────────────────────────┐
│                                                            │
│  S — SPOOFING (Suplantación de Identidad)                 │
│                                                            │
│  ¿Alguien puede hacerse pasar por otro?                    │
│                                                            │
│  Ejemplo: Un atacante usa credenciales robadas para        │
│           acceder como un administrador.                   │
│                                                            │
│  Mitigación:                                               │
│    • Autenticación fuerte (MFA, certificados, biometría).  │
│    • Tokens de sesión con tiempo de vida corto.            │
│    • mTLS entre servicios (cada servicio tiene certificado).│
│    • JWT firmados y con audiencia (aud claim).             │
│                                                            │
├────────────────────────────────────────────────────────────┤
│                                                            │
│  T — TAMPERING (Manipulación de Datos)                     │
│                                                            │
│  ¿Pueden alterar datos en tránsito o en reposo?            │
│                                                            │
│  Ejemplo: Un atacante modifica el monto de un pedido       │
│           durante el tránsito entre frontend y backend.    │
│                                                            │
│  Mitigación:                                               │
│    • TLS 1.3 para datos en tránsito.                       │
│    • Firmas digitales / HMAC para integridad de mensajes.  │
│    • Validación de input en servidor (nunca confiar en     │
│      lo que el cliente envía).                             │
│    • Checksums en datos almacenados.                       │
│                                                            │
├────────────────────────────────────────────────────────────┤
│                                                            │
│  R — REPUDIATION (Repudio / Negación)                      │
│                                                            │
│  ¿Pueden negar haber realizado una acción?                 │
│                                                            │
│  Ejemplo: Un cliente niega haber hecho una compra de $500. │
│           Un administrador borra datos y niega haber sido. │
│                                                            │
│  Mitigación:                                               │
│    • Logs de auditoría inmutables (append-only logs).      │
│    • Firmas digitales en transacciones críticas.           │
│    • Blockchain o WORM storage para registros financieros. │
│    • Cada acción atribuible a un usuario específico.       │
│                                                            │
├────────────────────────────────────────────────────────────┤
│                                                            │
│  I — INFORMATION DISCLOSURE (Divulgación de Información)   │
│                                                            │
│  ¿Pueden ver datos que no deberían ver?                    │
│                                                            │
│  Ejemplo: API devuelve datos de más en la respuesta.       │
│           Stack traces en producción revelan estructura.   │
│                                                            │
│  Mitigación:                                               │
│    • Output filtering: solo devolver lo necesario.         │
│    • DTOs específicos por endpoint (no entidades JPA).     │
│    • Manejo de errores genérico en producción.             │
│    • Datos sensibles enmascarados en logs.                 │
│                                                            │
├────────────────────────────────────────────────────────────┤
│                                                            │
│  D — DENIAL OF SERVICE (Denegación de Servicio)            │
│                                                            │
│  ¿Pueden tumbar el servicio o degradarlo?                  │
│                                                            │
│  Ejemplo: 10,000 requests/segundo saturan la API.          │
│           Upload de archivo de 10GB agota el disco.        │
│                                                            │
│  Mitigación:                                               │
│    • Rate limiting por IP, usuario, API key.               │
│    • Límites de tamaño en uploads.                         │
│    • Auto-scaling + DDoS protection (AWS Shield, CF).      │
│    • Timeouts en todas las operaciones.                    │
│                                                            │
├────────────────────────────────────────────────────────────┤
│                                                            │
│  E — ELEVATION OF PRIVILEGE (Escalación de Privilegios)    │
│                                                            │
│  ¿Pueden obtener más permisos de los que deberían?         │
│                                                            │
│  Ejemplo: Un usuario normal modifica la URL de admin       │
│           y accede a funciones administrativas.            │
│           Un contenedor escapa y accede al host.            │
│                                                            │
│  Mitigación:                                               │
│    • Principio de mínimo privilegio (IAM, RBAC).           │
│    • Separación de roles (admin vs usuario).                │
│    • Contenedores sin root (USER 1000 en Dockerfile).      │
│    • SecurityContext en Kubernetes (readOnlyRootFilesystem).│
│    • Revisión periódica de permisos (IAM Access Analyzer). │
│                                                            │
└────────────────────────────────────────────────────────────┘
```

### Ejercicio STRIDE en tu Sistema

```
Toma un diagrama de tu sistema (C4 Nivel 2, Contenedores). Para cada flecha
entre componentes, pregúntate las 6 preguntas STRIDE.

Componente A ──► Componente B
  (API Node)     (PostgreSQL)

S: ¿Alguien puede suplantar al API Node ante PostgreSQL?
   → Mitigación: credenciales en Secrets Manager, rotación automática.

T: ¿Alguien puede modificar los datos entre el API y PostgreSQL?
   → Mitigación: TLS para conexiones a BD (ssl_mode=require).

R: ¿El API puede negar haber ejecutado un INSERT?
   → Mitigación: audit logs con application user ID en cada query.

I: ¿PostgreSQL revela más datos de los necesarios al API?
   → Mitigación: SELECT solo columnas necesarias, no SELECT *.

D: ¿Pueden saturar la conexión entre el API y PostgreSQL?
   → Mitigación: connection pooling con límite máximo.

E: ¿Un atacante que compromete el API puede hacer DROP DATABASE?
   → Mitigación: usuario de BD sin permisos DDL en producción.
```

---

## 16.6 Secrets Management — El Arte de Guardar Secretos

### Lo Que El 90% de los Desarrolladores Hace Mal

```javascript
// ❌ ESTO EXISTE EN PRODUCCIÓN EN MILES DE EMPRESAS AHORA MISMO

// Opción 1: Hardcodeado en código
const STRIPE_SECRET = "sk_live_51H3j4k..."

// Opción 2: En archivo de configuración commiteado
// application.yml en el repositorio:
stripe:
  secret: sk_live_51H3j4k...

// Opción 3: En variable de entorno... que se imprime en logs
console.log("Starting with config:", process.env); // OOPS
// [2024-01-15 10:30:00] Starting with config: { STRIPE_SECRET: "sk_live_...", ... }

// Opción 4: En Dockerfile (capas inmutables, para siempre en la imagen)
ENV STRIPE_SECRET=sk_live_51H3j4k...
```

**Resultado**: El secret está en el código fuente, en el historial de git, en la imagen de Docker, en los logs de CloudWatch, y potencialmente en el laptop de cada desarrollador. Una brecha, 50 lugares donde se filtró.

### La Arquitectura Correcta

```
┌──────────────────────────────────────────────────────────────────┐
│                                                                   │
│  ┌──────────┐     ┌─────────────────┐     ┌──────────────────┐  │
│  │ Servicio │────►│ Secrets Manager │────►│ Recurso Protegido│  │
│  │          │     │ (Vault / AWS SM)│     │ (BD, API Key...) │  │
│  └──────────┘     └─────────────────┘     └──────────────────┘  │
│                         │                                         │
│                         │ El secret NUNCA está en código.         │
│                         │ Se obtiene en RUNTIME.                  │
│                         │ Se puede ROTAR sin redeploy.            │
│                         │ Se audita cada acceso.                  │
│                                                                   │
└──────────────────────────────────────────────────────────────────┘
```

```java
// ✅ Cómo se ve en la práctica

// En desarrollo local: variables de entorno (ok para dev)
// En producción: Secrets Manager / Vault

@Service
public class StripeService {
    private final String apiKey;

    // El secret se obtiene AL INICIAR la aplicación, desde Secrets Manager
    public StripeService(SecretsManagerClient secrets) {
        this.apiKey = secrets.getSecret("stripe/api-key");
    }
    
    public PaymentResult charge(PaymentRequest request) {
        Stripe.apiKey = this.apiKey;
        // ... usar Stripe
    }
}

// La aplicación NUNCA ve la API key en su código fuente.
// El secret se obtiene en runtime de Secrets Manager.
// Se puede rotar la API key en Stripe → actualizar Secrets Manager →
// reiniciar pods → nuevo secret en uso. Cero cambios de código.
```

### Checklist Anti-Filtración

```
[ ] git-secrets o truffleHog en pre-commit hooks.
[ ] Escaneo de secretos automatizado en CI/CD.
[ ] Si commiteaste un secreto: ROTARLO INMEDIATAMENTE.
    (Eliminarlo del commit no sirve: sigue en el historial de git)
[ ] Variables de entorno inyectadas en runtime (K8s Secrets,
    AWS Parameter Store, Vault Agent Injector).
[ ] NUNCA loguear variables de entorno completas.
[ ] Sanitización de logs: enmascarar campos como "password",
    "secret", "token", "key", "authorization".
```

---

## 16.7 Seguridad en la Cadena de Suministro

### Tu Aplicación Es un 90% de Código que No Escribiste

```
Una aplicación Node.js típica:
  ┌──────────┐
  │ Tu código│ ← 10% (5,000 líneas)
  └──────────┘
  ┌──────────┐
  │node_modul│ ← 90% (500,000 líneas de dependencias)
  │es/       │    Cada dependencia es código de TERCEROS
  │          │    ejecutándose con tus permisos.
  └──────────┘
```

### Lo que Debes Tener

```
SBOM — Software Bill of Materials
Una lista de CADA dependencia, versión, licencia y hash criptográfico
de tu aplicación. Es como la lista de ingredientes de un producto
alimenticio. Si se descubre una vulnerabilidad en log4j, puedes
saber en 5 minutos si TU sistema está afectado.

Herramientas:
  • Syft (genera SBOM)
  • Grype (escanea SBOM por vulnerabilidades)
  • Dependabot (GitHub, monitorea dependencias)
  • Snyk (monitorea + corrige)
  • Trivy (escanea imágenes Docker)
  • OWASP Dependency Check
```

```bash
# Generar SBOM para un proyecto
syft packages my-app:latest -o spdx-json > sbom.json

# Escanear la imagen Docker por vulnerabilidades
trivy image my-app:latest

# Integrar en CI/CD: bloquear el deploy si hay CRITICAL
trivy image --severity CRITICAL --exit-code 1 my-app:latest
```

---

## 16.8 Security Checklist del Arquitecto (Pre-Deploy)

```
ANTES de que cualquier código vaya a producción, verifica:

[ ] ¿Todos los endpoints requieren autenticación?
    (Excepto los explícitamente públicos: login, register, health)

[ ] ¿Todas las conexiones externas usan TLS 1.2+?

[ ] ¿Las conexiones INTERNAS también usan TLS/mTLS?
    (Zero trust: la red interna no es segura)

[ ] ¿Los passwords se almacenan con bcrypt/argon2?
    (NUNCA texto plano, NUNCA MD5, NUNCA SHA sin salt)

[ ] ¿Los datos de tarjeta NUNCA tocan tus servidores?
    (Tokenización vía iframe del proveedor de pagos)

[ ] ¿Las claves de API, secrets, passwords están en Secrets Manager?

[ ] ¿Hay rate limiting en todos los endpoints públicos?

[ ] ¿Los uploads de archivo tienen límite de tamaño y tipo?

[ ] ¿El CORS está configurado explícitamente (sin wildcard *)?

[ ] ¿Los logs NO contienen datos sensibles (passwords, tokens, tarjetas)?

[ ] ¿Hay auditoría para acciones críticas (cambios de rol, eliminaciones)?

[ ] ¿Las imágenes Docker no ejecutan como root?

[ ] ¿Los puertos de debug/management no están expuestos?

[ ] ¿Se ejecutó SAST + SCA + escaneo de imagen en el pipeline?

[ ] ¿Hay un plan de respuesta a incidentes documentado?
```

---

> **Reflexión del capítulo**: La seguridad no es una feature que añades al final. Es una propiedad emergente de cada decisión arquitectónica que tomas. Un sistema seguro no es el que tiene más firewalls, es el que fue diseñado asumiendo que CADA capa fallará. La pregunta no es "¿somos seguros?". La pregunta es "¿qué pasa CUANDO una capa falle?"
