# Capítulo 36: Catálogo de Anti-Patrones Arquitectónicos

> "Aprende de los errores de otros. No vivirás lo suficiente para cometerlos todos tú mismo."

## 36.1 ¿Qué es un Anti-Patrón y Por Qué Debes Conocerlos?

Un anti-patrón es una solución que parece buena en el momento pero que consistentemente produce malos resultados. Son las "mejores prácticas" al revés: trampas comunes en las que caen arquitectos de todos los niveles.

**Saber reconocer un anti-patrón ANTES de implementarlo es una de las habilidades más valiosas del arquitecto.** Es más fácil evitar un pantano que salir de él.

Aquí catalogamos los anti-patrones más destructivos que he visto (y cometido) en 20 años. Para cada uno explico: qué es, por qué parece buena idea, por qué es terrible, cómo detectarlo, y cómo salir de él si ya caíste.

---

## 36.2 Anti-Patrones de Diseño de Sistema

### 1. Big Ball of Mud (Gran Bola de Barro)

```
¿Qué es?
Un sistema sin arquitectura discernible. Todo conectado con todo.
Cualquier cambio rompe algo en el otro extremo del sistema.

¿Por qué pasa?
- Nunca hubo un arquitecto (o fue ignorado).
- Presión de time-to-market: "luego lo arreglamos" (nunca lo arreglaron).
- Alta rotación de equipo: cada dev nuevo añade más barro.
- "El código es auto-documentado" (mentira).

Síntomas:
  • Tiempo de onboarding >3 meses ("solo Juan entiende el sistema").
  • Cada sprint: 30% features, 70% arreglar roturas.
  • Tests inexistentes o inútiles.
  • "No podemos actualizar la versión de X porque rompe todo."

Cómo evitarlo:
  • Arquitectura desde el día 1 (aunque sea simple).
  • Boy Scout Rule estricta: deja el código mejor de lo que lo encontraste.
  • Dedica 20% de cada sprint a mejora técnica. Sin excusas.

Cómo salir:
  • Strangler Fig Pattern (Cap 26): reconstruir por partes.
  • Prioriza los módulos que más duelen.
  • No intentes un "big rewrite" — puede matar la empresa.
```

### 2. Architecture by Implication (Arquitectura por Omisión)

```
¿Qué es?
"No necesitamos documentación. El equipo es bueno. La arquitectura
 surgirá naturalmente del código."

¿Por qué parece buena idea?
  • "Somos ágiles, no necesitamos diseño."
  • "Los ADRs son burocracia."
  • "Confiamos en nuestro senior developer."

¿Por qué es terrible?
  • 6 meses después: nadie sabe por qué se tomó la decisión X.
  • El senior developer renuncia y se lleva el conocimiento.
  • Módulos inconsistentes porque cada dev interpretó "lo correcto" diferente.
  • Imposible onboardear nuevos devs.

Síntomas:
  • Preguntas en Slack: "¿por qué usamos MongoDB aquí y PostgreSQL allá?"
    Respuesta: silencio, o "no sé, así estaba cuando llegué".
  • Cero ADRs. Cero diagramas actualizados.
  • Cada vez que alguien pregunta "¿cuál es nuestra arquitectura?"
    la respuesta es vaga: "microservicios... más o menos."

Cómo evitarlo:
  • ADRs desde la primera decisión. No negocies esto.
  • Diagrama C4 de contexto y contenedores (mínimo).
  • Documentar NO es opcional. Es parte de la definición de "done".

Cómo salir:
  • Reverse-engineer ADRs: entrevista a los devs más antiguos,
    documenta lo que recuerdan.
  • Sesión de "archaeología": analiza el código para entender la
    arquitectura real (no la imaginada).
  • Escribe ADRs retrospectivos: "ADR-001: Por qué tenemos MongoDB
    (según los que estaban aquí en 2021)"
```

### 3. Ivory Tower Architecture (Arquitecto de Torre de Marfil)

```
¿Qué es?
Un arquitecto que diseña en aislamiento, entrega diagramas perfectos
al equipo, y nunca toca código ni enfrenta las consecuencias de sus
decisiones.

¿Por qué parece buena idea?
  • "El arquitecto debe tener visión global sin distraerse con detalles."
  • "Para eso están los developers, para implementar."
  • División del trabajo parece eficiente.

¿Por qué es terrible?
  • Los diagramas no sobreviven al primer sprint de implementación.
  • El arquitecto diseña soluciones para problemas que no existen.
  • El equipo resiente las decisiones impuestas ("ellos vs nosotros").
  • El arquitecto pierde relevancia técnica gradualmente.

Síntomas:
  • "El diagrama dice X, pero implementarlo así es imposible."
  • El arquitecto no sabe qué versión de Java/Node usan en producción.
  • Los devs implementan "mal" a propósito para forzar un cambio.
  • ADRs que dicen "usaremos X" pero nadie del equipo fue consultado.

Cómo evitarlo:
  • Arquitecto dedica ≥20% del tiempo a codificar en producción.
  • Las decisiones arquitectónicas se toman CON el equipo, no PARA el equipo.
  • El arquitecto participa en guardias y siente el dolor de sus decisiones.
  • ADRs requieren "consultados:" antes de ser aceptados.

Cómo salir:
  • Arquitecto: pide feedback explícito al equipo.
  • Arquitecto: agenda pair programming con cada team lead.
  • Arquitecto: trabaja en un bug de producción este sprint.
```

---

## 36.3 Anti-Patrones de Microservicios

### 4. Distributed Monolith (Monolito Distribuido)

```
¿Qué es?
Microservicios que solo funcionan si todos están desplegados juntos.
Comparten base de datos, requieren coordinación de releases,
y un fallo en uno tumba a los demás.

¿Por qué pasa?
- Migraste de monolito a microservicios sin rediseñar.
- Compartir BD era "más fácil" que implementar APIs.
- Los servicios se llaman sincrónicamente en cadena (A→B→C→D→E).

Síntomas:
  • Despliegas "todos los microservicios" juntos cada sprint.
  • "No podemos desplegar Pedidos sin desplegar Pagos también."
  • Varios servicios escriben en la misma tabla de BD.
  • Un timeout en servicio D causa cascada de errores hasta A.

Cómo evitarlo:
  • Database per Service (sin excepciones).
  • Comunicación asíncrona donde sea posible.
  • Diseña para fallos parciales desde el día 1.
  • Si no puedes desplegar un servicio sin los demás, es un monolito.

Cómo salir:
  • Identifica las dependencias más acopladas.
  • Introduce eventos para desacoplar (event-driven).
  • Migra datos a schemas/Bds separadas gradualmente.
  • Si todo está demasiado acoplado, reconsidera volver a un
    monolito modular (mejor monolito que distributed monolith).
```

### 5. Nanoservices (Servicios Atómicos)

```
¿Qué es?
Llevar microservicios al extremo: un servicio por cada función.
50 "microservicios" para lo que deberían ser 5.

¿Por qué parece buena idea?
  • "Es lo que hace Netflix." (No a tu escala.)
  • "Cada función es independiente."
  • "Single Responsibility Principle llevado al extremo."

¿Por qué es terrible?
  • Latencia de red por todas partes (una operación = 10 llamadas).
  • Costo operativo enorme (50 servicios que monitorear, desplegar).
  • Imposible debuggear (un request toca 20 servicios).
  • Cada servicio tiene overhead de boilerplate y configuración.

Síntomas:
  • Servicios de 50 líneas de código + 500 líneas de configuración.
  • Equipo de 5 personas "dueño" de 15 microservicios.
  • El diagrama de arquitectura parece una sopa de nodos.

Cómo evitarlo:
  • Regla: un microservicio por bounded context, no por función.
  • Si tu servicio no tiene su propia base de datos, probablemente
    no debería ser un servicio separado.
  • Pregúntate: ¿este servicio necesita escalar independientemente?
    Si no → es candidato a ser parte de su bounded context.

Cómo salir:
  • Fusionar nanoservices en servicios de bounded context.
  • Si comparten dominio, comparten deployable.
  • Menos servicios = menos latencia = menos operaciones = más felicidad.
```

### 6. Shared Database (Base de Datos Compartida)

```
¿Qué es?
Múltiples microservicios leen y escriben en la misma base de datos.
El peor anti-patrón de microservicios. El más común.

¿Por qué parece buena idea?
  • "Es más fácil que migrar los datos."
  • "Necesito datos del servicio de Usuarios en mi servicio de Pedidos."
  • "Mantener dos BDs es demasiado trabajo."

¿Por qué es terrible?
  • Acoplamiento extremo: si Pedidos cambia un schema, Usuarios se rompe.
  • Imposible escalar BDs independientemente.
  • Imposible elegir la BD correcta para cada servicio
    (todos atrapados en PostgreSQL aunque Analytics necesite Elasticsearch).
  • Un query malo de Analytics tumba las escrituras de Pedidos.

Ejemplo real de mi carrera:
  Servicio de Reportes hacía SELECT * sin WHERE a las 3 PM.
  La tabla de pedidos (100M rows) se lockeaba.
  Los usuarios no podían comprar durante 20 minutos.
  Causa raíz: BD compartida.

Cómo evitarlo:
  • Database per Service. Siempre. Es la regla más importante de microservicios.
  • Los servicios se comunican vía API, no vía SQL.
  • Si necesitas datos de otro servicio: llama a su API, no a su BD.

Cómo salir:
  • Identifica qué tablas pertenecen a cada servicio.
  • Extrae las tablas a BDs separadas.
  • Reemplaza joins entre servicios por API calls o eventos.
  • Usa CDC (Debezium) para replicar datos que realmente necesitas
    (pero cada servicio es dueño de su copia).
```

---

## 36.4 Anti-Patrones de Datos

### 7. God Table (Tabla Dios)

```
¿Qué es?
Una tabla con 100+ columnas que intenta contener toda la información
de una entidad. "users" con address1, address2, city, state, zip,
country, phone1, phone2, email1, email2, social_security,
mother_maiden_name, blood_type, favorite_color, last_5_orders,
loyalty_points, marketing_preferences_1, marketing_preferences_2... 

Síntomas:
  • "ALTER TABLE users ADD COLUMN..." ocurre cada sprint.
  • Queries que solo usan 5 de 120 columnas.
  • Índices que cubren combinaciones extrañas de columnas.
  • Tiempo de backup de una tabla es 10x más que todas las demás juntas.

Cómo evitarlo:
  • Normaliza. Las tablas auxiliares existen por una razón.
  • Si una entidad tiene subtipos, usa herencia de tablas o JSONB.
  • Pregunta: ¿estas columnas cambian juntas? → Misma tabla.
               ¿Cambian independientemente? → Tablas separadas.
```

### 8. Entity Service Anti-Pattern

```
¿Qué es?
Crear un microservicio CRUD por cada entidad de la base de datos.
"UserService", "ProductService", "OrderService", "AddressService".

¿Por qué es un problema?
Dividir por entidad = dividir por tabla = acoplado a la estructura
de datos, no al dominio del negocio. Un "crear pedido" requiere
llamar a 4 servicios (UserService, ProductService, OrderService,
AddressService), y si alguno falla, todo falla.

Cómo evitarlo:
  • Divide por bounded context (dominio), no por entidad.
  • Un servicio de "Pedidos" que maneja el flujo completo.
  • Piensa en procesos de negocio, no en tablas de BD.
```

---

## 36.5 Anti-Patrones de Comunicación

### 9. Death Star (Estrella de la Muerte)

```
¿Qué es?
Un sistema donde todos los servicios se llaman entre sí directamente.
El diagrama de dependencias parece la Estrella de la Muerte:
un enredo de líneas de un nodo a otro.

¿Por qué pasa?
  • Crecimiento orgánico sin gobernanza.
  • "Solo una llamada rápida a ese servicio" repetido 500 veces.
  • No hay API Gateway ni event bus.

Síntomas:
  • "¿Qué servicios necesito desplegar para esta feature?" → silencio.
  • Timeout en servicio Z causa errores en A, B, C, D, y E.
  • Imposible hacer un mapa de dependencias actualizado.

Cómo evitarlo:
  • API Gateway como punto único de entrada.
  • Event bus para desacoplar notificaciones.
  • Service Mesh para visibilidad de dependencias.
  • Limitar llamadas síncronas en cadena a máximo 1-2 hops.

Cómo salir:
  • Implementar API Gateway como fachada.
  • Migrar notificaciones a eventos.
  • Identificar y eliminar dependencias circulares.
  • Usar un service mesh (Istio) para visibilidad y control.
```

### 10. Infinite Retry Loop (Bucle de Reintentos Infinito)

```
¿Qué es?
Servicio A reintenta llamar a B. B reintenta llamar a C. C falla.
B reintenta 5 veces. Cada reintento de B causa un reintento de A.
Resultado: la falla se amplifica.

Servicio A ──► Servicio B ──► Servicio C (CAÍDO)
   │                │
   │ reintenta      │ reintenta
   │ 3 veces        │ 5 veces
   ▼                ▼
  (15 llamadas     (5 llamadas
   totales a B)     a C ya caído)

Esto escala: 100 requests iniciales × 15 reintentos = 1,500
llamadas a un servicio que ya está sufriendo.

Cómo evitarlo:
  • Circuit Breaker: después de N fallos, no reintentes más.
  • Backoff exponencial con jitter.
  • Límite total de reintentos por request (máximo 2-3).
  • Timeouts que consideren la cadena completa.
```

---

## 36.6 Anti-Patrones de Seguridad

### 11. Hardcoded Secrets (Secretos en Código)

```
¿Qué es?
API keys, contraseñas, tokens hardcodeados en el código fuente.

Ejemplos reales que he visto:
  • aws_access_key_id en un archivo application.yml commiteado.
  • password de BD en una variable de entorno de Dockerfile
    (que está en el repo).
  • Token de Stripe en una constante de JavaScript.
  • "admin/admin" como credenciales de staging... en producción.

Esto no es un error de novato que "le pasa a otros".
En 2023, GitHub detectó 12.8 MILLONES de secretos commiteados.
Es el anti-patrón de seguridad #1.

Solución:
  • Secrets Manager / Vault para TODO.
  • Pre-commit hooks que escanean secretos (git-secrets, truffleHog).
  • Variables de entorno inyectadas en runtime, no en código.
  • Rotación automática de credenciales.
  • Si commiteaste un secreto: rotarlo INMEDIATAMENTE (no basta con
    eliminarlo del repo, ya está en el historial de git).
```

### 12. No Defense in Depth (Una Sola Capa de Seguridad)

```
¿Qué es?
Confiar en que "el firewall nos protege" o "la VPN es segura".
Una sola barrera. Cuando cae, todo está expuesto.

Ejemplo: "Estamos en la VPC, no necesitamos TLS entre servicios."
         Un atacante obtiene acceso a UN contenedor.
         Ahora puede sniffear todo el tráfico interno en texto plano.

Solución: Zero trust. Nada es seguro por defecto.
  • TLS/mTLS para toda comunicación (interna y externa).
  • Auth en cada endpoint (incluso internos).
  • Mínimo privilegio en IAM (no "*" en policies).
  • Segmentación de red (security groups restrictivos).
```

---

## 36.7 Anti-Patrones Organizacionales

### 13. Silo Teams (Equipos en Silos por Tecnología)

```
Ya lo vimos en el Capítulo 34, pero como anti-patrón:
Equipo Frontend, Equipo Backend, Equipo DBA.

Consecuencia: Cada feature requiere coordinación de 3 equipos.
              Backlog separado, prioridades conflictivas.
              Sprint de Frontend no se alinea con Backend.
              "Nosotros ya terminamos, el problema es Backend."

Solución: Stream-aligned teams con full-stack capabilities.
```

### 14. Framework Obsession (Obsesión con Frameworks)

```
¿Qué es?
Elegir tecnologías solo porque son populares/modernas, sin evaluar
si resuelven el problema.

Ejemplos:
  • Kubernetes para una app de 2 servicios (ECS/Cloud Run bastaba).
  • Kafka para 100 eventos/día (Redis Streams o SQS bastaba).
  • Microservicios para un equipo de 3 personas.
  • Machine Learning para "if score > 60 → approved else → denied".

Síntomas:
  • "Lo usamos porque [FAANG] lo usa."
  • Complejidad innecesaria justificada como "preparación para escala".
  • El equipo pasa más tiempo configurando la herramienta que
    construyendo producto.

Prevención:
  • ADR con opciones evaluadas (no solo la que está de moda).
  • "¿Cuál es la opción más simple que funciona?" como primera pregunta.
  • Prueba de concepto antes de adoptar.
  • Trátalo como experimento, no como dogma.
```

---

## 36.8 Cómo Detectar Anti-Patrones Temprano

### Olores Arquitectónicos (Architecture Smells)

| Olor | Pregunta de Diagnóstico |
|------|------------------------|
| **Rigidez** | ¿Un cambio simple afecta múltiples módulos? |
| **Fragilidad** | ¿Cambios en módulo A rompen módulo Z sin relación? |
| **Inmovilidad** | ¿No puedes reutilizar ningún componente en otro proyecto? |
| **Viscosidad** | ¿Es más fácil hacer un hack que seguir la arquitectura? |
| **Opacidad** | ¿Nadie entiende el sistema completo? |
| **Deuda rampante** | ¿La deuda técnica crece cada sprint? |

### Health Check Trimestral de Arquitectura

```
Responde honestamente (1 = terrible, 5 = excelente):

[ ] Entendemos nuestra arquitectura actual.
[ ] La arquitectura documentada refleja la realidad del código.
[ ] Los nuevos devs son productivos en <1 mes.
[ ] Podemos desplegar un cambio sin miedo.
[ ] Los fallos son aislados (no en cascada).
[ ] Podemos cambiar una BD sin reescribir todo.
[ ] La seguridad no depende de una sola barrera.
[ ] Los equipos pueden trabajar sin bloqueos entre sí.

Puntuación < 20: Tienes trabajo urgente que hacer.
Puntuación 20-30: Bien, pero hay áreas de mejora claras.
Puntuación > 30: Excelente. Sigue así y no bajes la guardia.
```

---

> **Reflexión del capítulo**: Los anti-patrones son como las enfermedades: más vale prevenir que curar. Pero si ya estás enfermo (ya tienes un Big Ball of Mud, un Distributed Monolith, o una God Table), no te desesperes. Todos los sistemas tienen anti-patrones en algún grado. La diferencia entre un buen arquitecto y uno excelente no es nunca cometer errores — es reconocerlos temprano y tener la valentía de corregirlos. No te enamores de tu arquitectura. Enamórate de resolver problemas.
