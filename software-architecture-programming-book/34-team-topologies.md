# Capítulo 34: Team Topologies — La Arquitectura de los Equipos

> "Si tienes un problema de arquitectura que no puedes resolver técnicamente, mira la estructura de tus equipos. Ahí está la respuesta."

## 34.1 ¿Por Qué un Arquitecto Debe Entender de Equipos?

Imagina que eres arquitecto en una empresa con 40 desarrolladores. Diseñaste microservicios hermosos: Catálogo, Pedidos, Pagos, Inventario. Diagramas perfectos, separación de responsabilidades impecable.

Pero la realidad es: tienes 40 personas en un solo equipo monolítico, con un solo backlog, un solo daily, y todas tocan todas las bases de código.

**El resultado**: en 6 meses, tus microservicios comparten base de datos porque "era más fácil". Las APIs se acoplan porque "Juan de pagos y María de pedidos hablaron y decidieron compartir tabla". Tu arquitectura diseñada murió en producción.

**La lección**: la arquitectura del software refleja la arquitectura de los equipos. No puedes diseñar una sin la otra.

Aquí vamos a entender **por qué** pasa esto, **cómo** diseñar equipos que produzcan buena arquitectura naturalmente, y **qué hacer** cuando la estructura actual te está frenando.

## 34.2 Conway's Law en Profundidad

> "Cualquier organización que diseña un sistema producirá un diseño cuya estructura es copia de la estructura de comunicación de la organización." — Melvin Conway, 1968

### ¿Qué significa esto realmente?

No es una metáfora. Es una observación empírica validada por décadas.

```
Organización:                    Sistema resultante:
┌──────────────────┐             ┌──────────────────┐
│   4 equipos      │             │  4 componentes    │
│   independientes │             │  que se comunican  │
│   que se comunican│            │  entre sí como     │
│   vía reuniones  │             │  los equipos       │
└──────────────────┘             └──────────────────┘
```

### Ejemplo concreto que verás en tu carrera

```
Escenario A: Empresa organizada por capa técnica
┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│Equipo        │  │Equipo        │  │Equipo        │
│Frontend      │  │Backend       │  │Base de Datos │
│(10 personas) │  │(20 personas) │  │(5 personas)  │
└──────┬───────┘  └──────┬───────┘  └──────┬───────┘
       │                 │                 │
       └─────────┬───────┴────────┬────────┘
                 │   Integración  │
                 ▼     DOLOROSA   ▼
          ┌────────────────────────────┐
          │  Sistema resultante:       │
          │  • API de frontend rígida  │
          │  • Backend monolítico      │
          │  • Base de datos            │
          │    desnormalizada           │
          │    (porque cada equipo      │
          │     optimiza su capa)       │
          └────────────────────────────┘


Escenario B: Empresa organizada por dominio de negocio
┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│Equipo        │  │Equipo        │  │Equipo        │
│Pedidos       │  │Catálogo      │  │Pagos         │
│(full-stack)  │  │(full-stack)  │  │(full-stack)  │
│5 personas    │  │4 personas    │  │4 personas    │
└──────┬───────┘  └──────┬───────┘  └──────┬───────┘
       │                 │                 │
       ▼                 ▼                 ▼
┌────────────┐  ┌────────────┐  ┌────────────┐
│Servicio    │  │Servicio    │  │Servicio    │
│Pedidos     │  │Catálogo    │  │Pagos       │
│+BD+UI      │  │+BD+UI      │  │+BD+UI      │
└────────────┘  └────────────┘  └────────────┘
```

El escenario B produce naturalmente una arquitectura de servicios desacoplados. No porque alguien diseñó un diagrama bonito, sino porque los equipos están estructurados para ello.

### La Trampa de Ignorar Conway

El error más común que cometen los arquitectos novatos (y muchos experimentados) es:

1. Diseñar la arquitectura ideal en un diagrama.
2. No cambiar la estructura de equipos.
3. Esperar que los equipos implementen fielmente el diagrama.

**Esto nunca funciona.** Los equipos optimizan para su propia comunicación. Si dos equipos necesitan coordinarse para cada feature, fusionarán sus componentes eventualmente. Si un equipo tiene dos dominios separados, los acoplará porque "total, somos el mismo equipo".

## 34.3 Los Cuatro Tipos Fundamentales de Equipos

Team Topologies (Matthew Skelton y Manuel Pais, 2019) define 4 tipos de equipos que necesitas entender:

### 1. Stream-Aligned Team (Equipo Alineado al Flujo)

El equipo principal. Dueño de un segmento completo del dominio de negocio.

```
┌─────────────────────────────────────────┐
│ Stream-Aligned Team: "Pedidos"          │
│                                          │
│ Responsable de TODO el flujo de pedidos: │
│  • UI de checkout                       │
│  • API de pedidos                       │
│  • Lógica de negocio                    │
│  • Base de datos de pedidos             │
│  • Integración con pagos                │
│  • Métricas y monitoreo                 │
│  • Despliegue y operación               │
│                                          │
│ Habilidades: Full-stack, autónomo.      │
│ Tamaño: 5-9 personas (Two-Pizza Rule).  │
└─────────────────────────────────────────┘
```

**Características**:
- Son dueños de un producto/servicio completo.
- Tienen todas las habilidades para entregar valor de forma independiente.
- Reciben features de Product Manager, entregan software funcionando.
- No dependen de otros equipos para su trabajo diario.

**Por qué importa**: Si tus equipos no son stream-aligned, tu arquitectura no puede ser de servicios independientes. Es físicamente imposible.

### 2. Enabling Team (Equipo Habilitador)

Equipo temporal que ayuda a los stream-aligned a adquirir nuevas capacidades.

```
Problema: Los equipos stream-aligned necesitan adoptar Kafka,
         pero ninguno tiene experiencia.

Solución: Enabling Team (2-3 expertos en Kafka).
          │
          ▼
┌──────────────────────────────────────────────┐
│ Enabling Team: "Streaming & Messaging"        │
│                                                │
│ Semana 1-2: Workshop con Equipo Pedidos       │
│ Semana 3-4: Pair programming, setup inicial   │
│ Semana 5-6: Documentación, templates, guías   │
│ Semana 7-8: El equipo ya vuela solo.          │
│                                                │
│ El enabling team se disuelve o pasa a otro     │
│ equipo que necesita ayuda.                     │
│                                                │
│ NO se quedan como dueños de Kafka.             │
│ El conocimiento se transfirió.                 │
└──────────────────────────────────────────────┘
```

**Anti-patrón**: El enabling team que se vuelve permanente y se convierte en cuello de botella. "Somos el equipo de Kafka, todo pasa por nosotros." Eso es un equipo complicado, no habilitador.

### 3. Complicated Subsystem Team (Equipo de Subsistema Complejo)

Para partes del sistema que requieren expertise muy especializado.

```
¿Cuándo necesitas uno?

Situación: Tu sistema de pagos necesita un motor de cálculo de
           impuestos para 5 países con regulaciones complejísimas.

ERROR común: Meter a 3 stream-aligned teams a aprender leyes
             fiscales. Resultado: 3 implementaciones inconsistentes.

CORRECTO: Crear un Complicated Subsystem Team dueño del motor
          de impuestos. 3-4 especialistas. Los demás equipos
          lo consumen como un servicio.
```

```
┌──────────────────────────────────────────────┐
│ Complicated Subsystem Team:                  │
│ "Tax Engine"                                 │
│                                               │
│ Dueños de:                                    │
│  • API de cálculo de impuestos                │
│  • Mantenimiento de reglas fiscales           │
│  • Actualización cuando cambian leyes         │
│                                               │
│ Otros equipos consumen la API.                │
│ Nadie más toca la lógica de impuestos.        │
│ Los especialistas están donde deben.          │
└──────────────────────────────────────────────┘
```

**Cuándo sí**: expertise muy especializado (criptografía, ML, compliance regulatorio, motor de recomendaciones).

**Cuándo no**: "porque el código legacy da miedo". Eso no es subsistema complejo, es deuda técnica.

### 4. Platform Team (Equipo de Plataforma)

El más importante para escalar. Construye la plataforma interna que acelera a los stream-aligned.

```
┌────────────────────────────────────────────────────┐
│ Platform Team                                       │
│                                                     │
│ Producto: "Internal Developer Platform"             │
│ Clientes: Stream-aligned teams                      │
│                                                     │
│ Provee (como servicio, no como permiso):            │
│  • CI/CD pipelines como templates                   │
│  • Infraestructura como código pre-hecha            │
│  • Monitoring, logging, alerting pre-configurados   │
│  • Guías de arquitectura + herramientas             │
│  • Service catalog + API gateway                    │
│                                                     │
│ Trata a los devs como tus clientes.                 │
│ Tu producto es su productividad.                    │
└────────────────────────────────────────────────────┘
```

**Principio fundamental**: La plataforma es un PRODUCTO, no un permiso. Los stream-aligned teams ELIGEN usar la plataforma porque les hace la vida más fácil, no porque son obligados.

## 34.4 Modos de Interacción entre Equipos

### Collaboration (Colaboración)
Trabajan juntos por un período para resolver un problema.

```
Ejemplo: Equipo Pedidos + Equipo Pagos colaboran 2 sprints
para rediseñar la integración del checkout.

  ┌──────────┐    ┌──────────┐
  │ Pedidos  │◄──►│  Pagos   │
  └──────────┘    └──────────┘
       │               │
       └─── Reuniones ─┘
            diarias
```

**Cuándo**: Exploración, innovación, rediseño de interfaces.
**Cuidado**: No mantener colaboración permanente (genera acoplamiento).

### X-as-a-Service
Un equipo consume lo que otro provee con expectativas claras.

```
Ejemplo: Equipo Pedidos consume API de Impuestos.

  ┌──────────┐    ┌──────────────┐
  │ Pedidos  │───►│ Tax Engine   │
  │(consumer)│    │(provider)    │
  └──────────┘    └──────────────┘

  • SLA definido (disponibilidad 99.9%, p95 < 50ms)
  • API documentada (OpenAPI)
  • Soporte: canal definido, no DMs al dev
```

### Facilitating
El enabling team ayuda sin construir ellos.

```
  ┌──────────────┐    ┌──────────┐
  │ Enabling     │───►│ Stream-  │
  │ Team (Kafka) │    │ Aligned  │
  └──────────────┘    └──────────┘
  
  El enabling team NO construye el servicio Kafka del equipo.
  Enseña, guía, revisa. El equipo construye.
```

## 34.5 Cómo Llegar a Esta Estructura desde Donde Estás

### Paso 1: Identifica tu Estructura Actual

```
¿Cómo está organizada tu empresa AHORA?

[ ] Por capa técnica (frontend, backend, DBA)
[ ] Por dominio (pedidos, pagos, catálogo)
[ ] Por feature/proyecto (equipos temporales)
[ ] Mixto / no hay estructura clara
[ ] Una sola "masa" de desarrolladores
```

### Paso 2: Identifica los Límites de Dominio

```
Ejercicio: Siéntate con el CTO/VP y respondan:

1. ¿Cuáles son los 3-5 dominios de negocio más importantes?
   Ej: Catálogo, Pedidos, Pagos, Envíos, Usuarios

2. Para cada uno: ¿podría un equipo de 5-8 personas ser dueño
   completo (UI, backend, datos, operaciones)?

3. ¿Qué dependencias entre dominios son inevitables?
   Ej: Pedidos siempre necesita leer datos de Catálogo.
```

### Paso 3: Diseña los Equipos Futuros

```
Estructura objetivo para una empresa de 40 devs:

Stream-Aligned Teams (core):
┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐
│Catálogo  │ │ Pedidos  │ │  Pagos   │ │  Envíos  │
│ 7 pers.  │ │ 8 pers.  │ │ 5 pers.  │ │ 4 pers.  │
└──────────┘ └──────────┘ └──────────┘ └──────────┘

Platform Team:
┌────────────────────────────────────────────┐
│ Developer Platform (8 personas)            │
│ CI/CD, Infra, Monitoreo, Seguridad Base    │
└────────────────────────────────────────────┘

Enabling Team (temporal, rotativo):
┌────────────────────────────────────────────┐
│ Data Engineering Enablement (3 personas)   │
│ Arquitectura + prácticas (2 personas)      │
└────────────────────────────────────────────┘

Complicated Subsystem (si necesario):
┌────────────────────────────────────────────┐
│ Compliance & Tax Engine (3 personas)       │
└────────────────────────────────────────────┘
```

### Paso 4: Migra Gradualmente

```
No reorganices 40 personas de golpe. Es trauma innecesario.

Fase 1 (Mes 1-2):
  • Identifica el dominio más independiente.
  • Crea el primer stream-aligned team (voluntarios).
  • Dales autonomía real.

Fase 2 (Mes 3-6):
  • Mueve al segundo y tercer dominio.
  • La plataforma existente se convierte en Platform Team.
  • Documenta el modelo para que todos entiendan el plan.

Fase 3 (Mes 7-12):
  • Completa la transición.
  • Disuelve los equipos por capa restantes.
  • Itera: los límites de equipo cambian con el tiempo.
```

### La Señal de que Funciona

```
✅ Los equipos despliegan independientemente (sin coordinar releases).
✅ Las dependencias entre equipos son vía APIs, no vía personas.
✅ Un equipo puede cambiar su stack sin afectar a otros.
✅ Nuevas personas son productivas en días, no meses.
✅ Los post-mortems son locales a un equipo, no involucran a 3.
```

## 34.6 Anti-Patrones de Estructura de Equipos

### 1. El Equipo "Utility" (Herramienta)

```
Problema: "Equipo de Base de Datos" que aprueba cada schema change.
          Cada PR que toca SQL espera 3 días su review.

Solución: Platform team que construye herramientas para que los
          equipos gestionen sus propias migraciones. Consultoría
          disponible, pero no gatekeeper.
```

### 2. El Equipo "Proxy" (Intermediario)

```
Problema: "Equipo de Integración" que está entre todos los demás.
          Todas las APIs pasan por ellos. Son cuello de botella.

Equipo A ──► Integración ◄── Equipo B
            (cuello de botella)

Solución: Los equipos se comunican directamente.
          API Gateway + contratos claros.
```

### 3. El "Super-Equipo" (Demasiado Grande)

```
Problema: 15 personas en un equipo. Standup de 30 minutos.
          Nadie sabe qué hace la mitad del equipo.

Solución: Máximo 7-9 personas (ideal 5-7).
          Si crece más, divide por sub-dominio.
```

### 4. El Equipo Fantasma (Sin Dueño)

```
Problema: El servicio de notificaciones. ¿Quién es dueño?
          "Ah, eso lo hizo Juan antes de irse."
          Nadie lo mantiene, nadie lo monitorea.

Solución: Toda pieza de software tiene un equipo dueño.
          Si no, no se despliega. Catálogo de servicios
          con owner explícito.
```

### 5. Los Dos Jefes (Reporte Compartido)

```
Problema: María reporta a la Gerente de Frontend y al
          Product Manager del equipo Pedidos. Prioridades
          conflictivas, estrés, código mediocre.

Solución: Reporte único al stream-aligned team.
          Las comunidades técnicas (frontend, backend)
          son guilds/comunidades, no jerarquías de reporte.
```

## 34.7 El Catálogo de Servicios — Mapa de tu Arquitectura

```
Cada servicio/producto interno debe tener:

• Nombre del servicio
• Descripción (1 párrafo)
• Equipo dueño (con canal de contacto)
• API documentation link
• Repositorio(s)
• SLAs (disponibilidad, latencia)
• Dependencias (qué consume, quién lo consume)
• Health dashboard link
• Runbooks (qué hacer si falla)
• On-call rotation
```

**Herramientas**: Backstage (Spotify), Cortex, OpsLevel, Port.

Un servicio sin owner documentado es deuda técnica andante.

## 34.8 Ejercicio Práctico del Capítulo

```
Toma tu empresa actual (o una que conozcas bien):

1. Dibuja cómo están organizados los equipos actualmente.
2. ¿Qué tipo(s) de equipo son? (stream-aligned, platform, etc.)
3. ¿La arquitectura del software refleja esta estructura?
   Da un ejemplo concreto.
4. ¿Hay un equipo "cuello de botella"? ¿Un equipo "fantasma"?
5. ¿Hay un Platform Team? Si no, ¿quién gestiona CI/CD, monitoring?
6. Rediseña: si pudieras reorganizar con Team Topologies,
   ¿cómo se verían los equipos? ¿Qué migrarías primero?
```

---

> **Reflexión del capítulo**: Llevo 20 años en esto y la lección más dura que aprendí es: los diagramas de arquitectura no valen nada sin los equipos correctos detrás. Puedes diseñar la arquitectura más elegante del mundo, pero si tus equipos no están organizados para soportarla, fracasará. El arquitecto que solo sabe de tecnología es medio arquitecto. El arquitecto completo entiende que el software lo construyen personas organizadas de cierta manera. Diseña tus equipos tan cuidadosamente como diseñas tus bases de datos.
