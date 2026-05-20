# Capítulo 3: Atributos de Calidad (Quality Attributes)

> "La calidad no es un acto, es un hábito." — Aristóteles

## 3.1 ¿Qué son los Atributos de Calidad?

Son las propiedades observables y medibles de un sistema que determinan qué tan bien satisface las necesidades de sus stakeholders. **No son funcionalidades**, son *cómo* se comporta el sistema.

La arquitectura existe principalmente para satisfacer atributos de calidad. Si todo fuera funcionalidad, no necesitarías arquitectura.

## 3.2 Taxonomía de Atributos de Calidad

### Rendimiento (Performance)
- **Latencia**: Tiempo para procesar una solicitud.
- **Throughput**: Solicitudes procesadas por unidad de tiempo.
- **Tiempo de respuesta bajo carga**: Percentiles (p50, p95, p99).
- **Eficiencia**: Recursos consumidos por operación.

> *Regla de oro*: Mide en percentiles, no en promedios. Un promedio oculta los outliers que matan la experiencia de usuario.

### Escalabilidad (Scalability)
- **Escalabilidad vertical (scale up)**: Máquina más grande.
- **Escalabilidad horizontal (scale out)**: Más máquinas.
- **Elasticidad**: Escalar automáticamente según demanda.
- **Coeficiente de escalabilidad**: ¿El throughput crece linealmente con los recursos?

### Disponibilidad (Availability)
- **Uptime**: Porcentaje de tiempo operativo (99.9% = 8.76h/año de downtime, 99.999% = 5.26min/año).
- **MTBF** (Mean Time Between Failures): Tiempo promedio entre fallos.
- **MTTR** (Mean Time To Recover): Tiempo promedio de recuperación.
- **Resiliencia**: Capacidad de recuperarse de fallos.

### Seguridad (Security)
- **Confidencialidad**: Solo usuarios autorizados acceden a los datos.
- **Integridad**: Los datos no se alteran sin autorización.
- **Disponibilidad**: El sistema está accesible cuando se necesita.
- **Autenticación, Autorización, Auditoría** (AAA).
- **No repudio**: Una acción no puede negarse después de realizarse.

### Mantenibilidad (Maintainability)
- **Modularidad**: Cambios aislados en componentes independientes.
- **Testabilidad**: Facilidad para verificar el comportamiento.
- **Comprensibilidad**: El código se entiende leyéndolo.
- **Deployabilidad**: Frecuencia y facilidad de despliegues.

### Otros Atributos Críticos

| Atributo | Definición |
|----------|-----------|
| **Usabilidad** | Facilidad de uso para el usuario final. |
| **Interoperabilidad** | Capacidad de integrarse con otros sistemas. |
| **Portabilidad** | Facilidad para migrar entre entornos. |
| **Observabilidad** | Capacidad de entender el estado interno desde afuera. |
| **Extensibilidad** | Capacidad de añadir funcionalidad sin modificar lo existente. |

## 3.3 Trade-offs entre Atributos

Ningún sistema maximiza todos los atributos simultáneamente. Cada decisión es un trade-off:

| Trade-off Clásico | Conflicto |
|-------------------|-----------|
| **Seguridad vs Rendimiento** | Encriptar todo añade latencia. |
| **Consistencia vs Disponibilidad** | Teorema CAP. |
| **Mantenibilidad vs Rendimiento** | Abstracciones añaden overhead. |
| **Escalabilidad vs Complejidad** | Microservicios escalan mejor, pero son más complejos. |
| **Time to Market vs Calidad** | Salir rápido implica deuda técnica. |

## 3.4 Estrategia para Priorizar Atributos

Usa el **Quality Attribute Workshop (QAW)**:

1. Identifica stakeholders (negocio, operaciones, desarrollo, usuarios).
2. Brainstorming de escenarios de calidad.
3. Votación y priorización.
4. Documenta como **escenarios concretos**:

```
Formato de escenario de calidad:
- Fuente de estímulo: Usuario final
- Estímulo: Realiza una búsqueda
- Artefacto: Servicio de búsqueda
- Entorno: Operación normal con 10k usuarios concurrentes
- Respuesta: Resultados devueltos
- Medida de respuesta: p95 < 200ms
```

## 3.5 Cómo Medir Cada Atributo (Porque Si No Se Mide, No Existe)

No basta con decir "el sistema debe ser rápido". Necesitas métricas concretas. Aquí te muestro cómo medir cada atributo de forma práctica:

### Midiendo Rendimiento

```
Métrica              Herramienta              Cómo lo mides
─────────────────────────────────────────────────────────────
p50, p95, p99        Prometheus + Grafana     Histograma de latencia por endpoint
latencia
Throughput            k6, wrk, Vegeta          Requests/segundo bajo carga
Tamaño de payload    APM (Datadog, NewRelic)   KB por response
Conexiones activas   Métricas de BD            Pool connections utilizado/máximo
Garbage collection   JMX, Prometheus           Pausas de GC, frecuencia

Ejemplo de PromQL (Prometheus Query Language):
# p95 de latencia por endpoint en los últimos 5 minutos
histogram_quantile(0.95,
  rate(http_request_duration_seconds_bucket[5m]))
  by (endpoint)
```

### Midiendo Disponibilidad

```
Disponibilidad = Tiempo total - Tiempo caído / Tiempo total

Ejemplo real:
En enero (31 días = 44,640 minutos):
  - Caída planificada (mantenimiento): 30 min → NO cuenta contra SLO
  - Caída no planificada (incidente): 45 min → SÍ cuenta
  - Disponibilidad: (44,640 - 45) / 44,640 = 99.899%

Los 9s que importan:
  99%     = 3.65 días/año caído     → Startup MVP
  99.9%   = 8.76 horas/año caído    → SaaS estándar
  99.95%  = 4.38 horas/año          → E-commerce serio
  99.99%  = 52.56 minutos/año       → Banca, salud
  99.999% = 5.26 minutos/año        → Telecom, infraestructura crítica

Cada 9 extra cuesta ~10x más en infraestructura.
```

### Midiendo Escalabilidad

```
Coeficiente de escalabilidad:
  Si duplicas recursos (2x servidores), ¿el throughput se duplica (2x)?

  Ideal: throughput crece linealmente (coeficiente = 1.0)
  Bueno: throughput crece sub-linealmente (coeficiente = 0.8-0.9)
  Malo: throughput apenas crece (coeficiente < 0.5) → cuello de botella

Cómo medirlo:
  Load test con 1, 2, 4, 8 instancias. Graficar throughput vs instancias.
  Si la curva se aplana, hay un cuello de botella (BD, lock, recurso compartido).
```

### Midiendo Mantenibilidad

```
Métrica                    Objetivo           Cómo medir
──────────────────────────────────────────────────────────
Tiempo de la suite CI     <10 min            Cronómetro del pipeline
Cobertura de código       >80%               JaCoCo, Istanbul, Coverlet
Complejidad ciclomática   <10 por método     SonarQube, CodeClimate
Deuda técnica (Sonar)     <5 días            SonarQube (Technical Debt ratio)
Frecuencia de deploys     Diario o más       DORA metrics
Tiempo de onboarding      <2 semanas         Encuesta a nuevos devs
```

## 3.6 ISO 25010: El Estándar Internacional

La ISO 25010 define el modelo de calidad de software más aceptado. No necesitas memorizarlo, pero sí conocer que existe y entender su estructura para hablar el mismo idioma que arquitectos enterprise y consultores.

```
ISO 25010 — Modelo de Calidad del Producto

┌──────────────────────────────────────────────────────┐
│                                                       │
│  Adecuación Funcional      Eficiencia de Desempeño   │
│  ├─ Completitud funcional  ├─ Comportamiento temporal │
│  ├─ Corrección funcional   ├─ Utilización de recursos │
│  └─ Pertinencia funcional  └─ Capacidad               │
│                                                       │
│  Compatibilidad            Usabilidad                 │
│  ├─ Coexistencia           ├─ Inteligibilidad          │
│  └─ Interoperabilidad      ├─ Aprendizaje              │
│                             ├─ Operabilidad            │
│  Fiabilidad                ├─ Protección error usuario │
│  ├─ Madurez                ├─ Estética de interfaz     │
│  ├─ Disponibilidad         └─ Accesibilidad            │
│  ├─ Tolerancia a fallos                                │
│  └─ Capacidad de recuperación                          │
│                                                       │
│  Seguridad                 Mantenibilidad             │
│  ├─ Confidencialidad       ├─ Modularidad              │
│  ├─ Integridad             ├─ Reusabilidad             │
│  ├─ No repudio             ├─ Analizabilidad           │
│  ├─ Responsabilidad        ├─ Modificabilidad          │
│  └─ Autenticidad           └─ Testabilidad             │
│                                                       │
│  Portabilidad                                          │
│  ├─ Adaptabilidad                                      │
│  ├─ Instalabilidad                                     │
│  └─ Reemplazabilidad                                   │
└──────────────────────────────────────────────────────┘
```

## 3.7 Cómo los Atributos Interactúan — Ejemplos Reales

### Ejemplo 1: Seguridad vs Rendimiento

```
Tu jefe pide: "Quiero TLS en todas las comunicaciones internas."

Decisión ingenua: "Ok, activamos TLS entre todos los microservicios."

Problema real:
  Sin TLS: 5000 req/s, p95 = 10ms
  Con TLS:  4500 req/s, p95 = 15ms  (10% menos throughput, 50% más latencia)

  El TLS handshake es costoso. Entre 50 microservicios, la latencia
  se acumula.

Solución real:
  • TLS con connection pooling (reutilizar conexiones TLS,
    no hacer handshake por request).
  • mTLS con certificados efímeros (SPIFFE).
  • Service Mesh (Istio) maneja TLS transparentemente.

Lección: Seguridad no es "activar o no activar".
         Es "activar con la estrategia correcta para minimizar impacto".
```

### Ejemplo 2: Consistencia vs Disponibilidad (Teorema CAP Explicado)

```
Imagina que tienes 3 nodos de base de datos replicados:

  ┌──────┐  ┌──────┐  ┌──────┐
  │Nodo A│  │Nodo B│  │Nodo C│
  │(NY)  │  │(LA)  │  │(LON) │
  └──────┘  └──────┘  └──────┘

Un usuario en NY actualiza su perfil (nombre: "Ana" → "Anita").

Escenario: El cable entre NY y LA se corta (partición de red).

┌──────────────────────────────────────────────────────┐
│ Tienes que elegir (el teorema CAP lo exige):         │
│                                                       │
│ CONSISTENCIA (CP):                                    │
│ "El usuario de LA NO puede ver el perfil hasta que    │
│  el cable se repare."                                 │
│                                                       │
│  ✓ Los datos son correctos siempre.                   │
│  ✗ El usuario de LA ve "Ana" aunque ya cambió.       │
│    O peor: no puede ver NADA porque rechazamos        │
│    lecturas inconsistentes.                           │
│                                                       │
│  Tecnologías CP: HBase, MongoDB (configurado),        │
│                  CockroachDB.                         │
│                                                       │
│ DISPONIBILIDAD (AP):                                  │
│ "El usuario de LA puede ver el perfil, aunque         │
│  vea 'Ana' en vez de 'Anita' hasta que sincronice."   │
│                                                       │
│  ✓ El sistema siempre responde.                       │
│  ✗ Datos temporalmente inconsistentes.               │
│                                                       │
│  Tecnologías AP: Cassandra, DynamoDB, CouchDB.        │
│                                                       │
│ PostgreSQL (sin réplicas): No está en esta disyuntiva │
│ porque no es distribuido. Tiene CA (Consistencia y    │
│ Disponibilidad) pero no P (Partition Tolerance).       │
└──────────────────────────────────────────────────────┘

¿Cuál elegir? Depende del caso de uso:
  • Saldo bancario: CP (prefiero "no disponible" a "saldo incorrecto").
  • Carrito de compras: AP (prefiero ver items viejos a no ver nada).
  • Timeline de Twitter: AP (eventual consistency aceptable).
```

### Ejemplo 3: Time to Market vs Mantenibilidad

```
Semana 1-4 (MVP):
  Decisión: "No implementamos Clean Architecture, vamos directo."
  Resultado: Lanzamos en 4 semanas. Clientes contentos. ✓

Semana 20:
  Decisión: "Los clientes piden features que tocan 5 módulos.
            Cada cambio rompe 3 tests y toma 3 días."
  Resultado: Velocidad = 20% de la semana 1.

Este es el trade-off más difícil de tu carrera:
  • Si priorizas calidad desde el día 1 → el MVP tarda 12 semanas.
    La startup se queda sin dinero.
  • Si priorizas velocidad → el MVP sale en 4 semanas.
    Pero en 6 meses no puedes moverte sin romper algo.

La respuesta no es blanco o negro. Es:
  1. MVP con velocidad (aceptando deuda controlada).
  2. Semana 5-8: pagar la deuda más urgente.
  3. De ahí en adelante: 80% features, 20% calidad.
  4. NUNCA dejar de pagar el 20%. Si lo haces, mueres en 12 meses.
```

## 3.8 Taller Práctico — Define los Atributos de Tu Sistema

```
Ejercicio: Toma un sistema que conozcas (o imagina uno nuevo)
y completa esta tabla.

Sistema: ___________________________________________

1. ¿Cuál es el ATRIBUTO #1 más importante?
   ¿Por qué? (Vincúlalo a una necesidad de negocio)

2. ¿Cuál es el #2?
   ¿Qué trade-off tienes que aceptar para lograrlo?

3. ¿Cuál es el MENOS importante?
   ¿Qué podrías sacrificar sin que el negocio sufra?

4. Para tu atributo #1, escribe un escenario concreto:
   - Fuente:
   - Estímulo:
   - Artefacto:
   - Entorno:
   - Respuesta esperada:
   - Medida de respuesta (número concreto):
```

### Ejemplo Resuelto

```
Sistema: API de Pedidos para E-commerce

1. Atributo #1: DISPONIBILIDAD
   Porque: Cada minuto caído = $800 en ventas perdidas.
           El CEO fue claro: "Prefiero lento que caído."

2. Atributo #2: RENDIMIENTO
   Trade-off: Implementamos multi-AZ + read replicas.
   Costo de infraestructura sube 40%. Aceptado por CFO.

3. Menos importante: EXTENSIBILIDAD
   Podemos hardcodear algunas reglas de negocio en V1
   si eso acelera el time to market.

4. Escenario concreto:
   - Fuente: Usuario final (App móvil)
   - Estímulo: Realiza checkout con 3 items
   - Artefacto: Servicio de Pedidos
   - Entorno: Black Friday, 10k usuarios concurrentes
   - Respuesta: Pedido creado y confirmado
   - Medida: p95 < 500ms, tasa de éxito > 99.5%
```

## 3.9 Anti-Patrones de Atributos de Calidad

- **Sobre-ingeniería**: Escalar para millones de usuarios cuando tienes cientos. Optimizas el atributo equivocado.
- **Optimización prematura**: Perder tiempo en micro-optimizaciones sin datos. Primero mide, luego optimiza.
- **Ignorar los -ilities**: Solo enfocarse en funcionalidades. Si el sistema es inmantenible, no importa qué tan bien funcione.
- **Maximizar un solo atributo**: "Necesitamos 100% de disponibilidad." Imposible. Y aunque fuera posible, el costo sería infinito.
- **Copiar atributos de Netflix**: Netflix necesita streaming global con 200M usuarios. Tú no (aún). Tus atributos de calidad son los de TU sistema, no los de otro.

---

> **Reflexión del capítulo**: Los atributos de calidad son el porqué de la arquitectura. Sin ellos, eres un desarrollador senior, no un arquitecto. Cada vez que tomes una decisión técnica, pregúntate: "¿qué atributo de calidad estoy optimizando y cuál estoy sacrificando?" Si no puedes responder, no has entendido tu propia decisión. Los atributos de calidad son el verdadero corazón de la arquitectura. Un sistema que funciona pero es lento, inseguro e inmantenible es un sistema fallido. Define tus atributos de calidad antes de escribir una sola línea de código.

---

← [Capítulo anterior](02-rol-arquitecto.md) | [Inicio](README.md) | [Capítulo siguiente →](04-principios-solid.md)
