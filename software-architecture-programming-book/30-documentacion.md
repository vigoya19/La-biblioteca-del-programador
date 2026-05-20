# Capítulo 30: Documentación de Arquitectura — C4, ADRs y Diagramas que Sirven

> "Un diagrama que nadie lee es peor que ningún diagrama. Genera falsa confianza."

## 30.1 El Problema con la Documentación Tradicional

La mayoría de la documentación de arquitectura sufre de:

- **Documentos de 200 páginas que nadie lee** después de la primera semana.
- **Diagramas obsoletos** que no reflejan la realidad del código (violación de "single source of truth").
- **Demasiado detalle** donde no se necesita y muy poco donde sí.
- **Formato inconsistente**: cada arquitecto usa su propio estilo, si es que documenta.
- **Sin audiencia clara**: el mismo diagrama intenta servir al dev junior y al CTO.

**Principio fundamental**: La documentación debe ser *fit for purpose* — lo suficientemente detallada para su audiencia, ni más ni menos.

## 30.2 El Modelo C4

Creado por Simon Brown. Organiza la documentación en 4 niveles de abstracción, como un Google Maps del software.

```
Nivel 1: System Context    (El país)     ← Para todos
Nivel 2: Container         (La ciudad)   ← Para técnicos y negocio
Nivel 3: Component         (El barrio)   ← Para desarrolladores
Nivel 4: Code              (Las calles)  ← Para el IDE (UML/ código)
```

### Nivel 1: Diagrama de Contexto

> **Audiencia**: Todos (técnicos y no técnicos).
> **Propósito**: ¿Qué sistema es este? ¿Con qué interactúa? ¿Quiénes lo usan?

```
┌──────────────────────────────────────────────────────────┐
│                                                           │
│   ┌─────────┐                          ┌──────────────┐  │
│   │ Cliente │                          │ Administrador│  │
│   │ (App    │                          │ (Panel Web)  │  │
│   │ Móvil)  │                          │              │  │
│   └────┬────┘                          └──────┬───────┘  │
│        │                                      │          │
│        │  "Realiza pedidos,                    │          │
│        │   consulta productos"                │          │
│        │                                      │          │
│        ▼                                      ▼          │
│   ┌──────────────────────────────────────────────────┐  │
│   │                                                  │  │
│   │          Sistema de E-Commerce                   │  │
│   │          [Software System]                       │  │
│   │                                                  │  │
│   └──────┬───────────────────────────────┬──────────┘  │
│          │                               │              │
│          │ "Procesa pagos"              │ "Envía       │
│          │                               │ emails"      │
│          ▼                               ▼              │
│   ┌────────────┐                   ┌────────────┐      │
│   │ Stripe     │                   │ SendGrid   │      │
│   │ [External  │                   │ [External  │      │
│   │  System]   │                   │  System]   │      │
│   └────────────┘                   └────────────┘      │
│                                                           │
└──────────────────────────────────────────────────────────┘

Título: System Context diagram for E-Commerce System
Descripción: Muestra el sistema, sus usuarios y sistemas externos.
```

**Reglas**:
- Tu sistema es UNA caja azul en el centro.
- Solo mostrar sistemas externos (cajas grises) y usuarios (figuras humanas).
- Cada flecha debe tener una intención (qué hace, no cómo).
- **Sin tecnología, sin protocolos, sin bases de datos.**

### Nivel 2: Diagrama de Contenedores

> **Audiencia**: Técnicos, stakeholders de negocio con conocimiento técnico.
> **Propósito**: ¿De qué está hecho el sistema? ¿Cómo se comunican las piezas grandes?

```
┌──────────────────────────────────────────────────────────────┐
│                                                               │
│  ┌─────────┐                                                 │
│  │ Cliente │                                                 │
│  │ (SPA)   │──── HTTPS ────┐                                 │
│  └─────────┘               │                                 │
│                             ▼                                 │
│  ┌──────────────────────────────────────────────────────┐    │
│  │              Single-Page Application                 │    │
│  │              [Container: React + Nginx]              │    │
│  └───────────────────────┬──────────────────────────────┘    │
│                          │ HTTPS (JSON)                       │
│                          ▼                                    │
│  ┌──────────────────────────────────────────────────────┐    │
│  │              API Application                          │    │
│  │              [Container: Node.js + Express]           │    │
│  │                                                       │    │
│  │  Lee/escribe ───►┌────────────┐                      │    │
│  │                  │ PostgreSQL │                      │    │
│  │                  │ [Container]│                      │    │
│  │                  └────────────┘                      │    │
│  │                                                       │    │
│  │  Cache ─────────►┌────────────┐                      │    │
│  │                  │   Redis    │                      │    │
│  │                  │ [Container]│                      │    │
│  │                  └────────────┘                      │    │
│  │                                                       │    │
│  │  Emite eventos ──► ┌──────────┐ ──► Consumidores    │    │
│  │                    │  Kafka   │                      │    │
│  │                    │[Container]│                     │    │
│  └────────────────────┴──────────┴──────────────────────┘    │
│                                                               │
└──────────────────────────────────────────────────────────────┘

Título: Container diagram for E-Commerce System
```

**Reglas**:
- Cada contenedor es una unidad desplegable independiente.
- Muestra: aplicaciones, bases de datos, sistemas de archivos, message brokers.
- Tecnologías específicas (Node.js, PostgreSQL, Kafka).
- Protocolos de comunicación (JSON/HTTPS, TCP, JDBC).

### Nivel 3: Diagrama de Componentes

> **Audiencia**: Desarrolladores, arquitectos.
> **Propósito**: ¿Cómo está estructurado un contenedor internamente?

```
┌───────────────────────────────────────────────────────┐
│         API Application [Container: Node.js]          │
│                                                        │
│  ┌──────────────────┐    ┌──────────────────┐        │
│  │ OrderController  │    │ProductController │        │
│  │ [Component:      │    │ [Component:      │        │
│  │  Router/Adapter] │    │  Router/Adapter] │        │
│  └────────┬─────────┘    └────────┬─────────┘        │
│           │                       │                  │
│           ▼                       ▼                  │
│  ┌──────────────────┐    ┌──────────────────┐        │
│  │PlaceOrderUseCase │    │SearchProductsUC  │        │
│  │ [Component:      │    │ [Component:      │        │
│  │  Use Case]       │    │  Use Case]       │        │
│  └────────┬─────────┘    └────────┬─────────┘        │
│           │                       │                  │
│           ▼                       ▼                  │
│  ┌──────────────────┐    ┌──────────────────┐        │
│  │ OrderRepository  │    │ProductRepository │        │
│  │ [Component:      │    │ [Component:      │        │
│  │  Port Interface] │    │  Port Interface] │        │
│  └────────┬─────────┘    └────────┬─────────┘        │
│           │                       │                  │
│           └───────────┬───────────┘                  │
│                       ▼                              │
│              ┌──────────────────┐                    │
│              │ PostgresAdapter  │                    │
│              │ [Component:      │                    │
│              │  Adapter]        │                    │
│              └──────────────────┘                    │
│                                                        │
└───────────────────────────────────────────────────────┘
```

### Nivel 4: Código

> **Audiencia**: Desarrolladores.
> **Propósito**: Diagrama de clases, secuencia, entidad-relación.

Este nivel se genera automáticamente del código (IDEs, herramientas UML). **No lo documentes a mano**, se desincroniza instantáneamente.

## 30.3 Notación y Herramientas

### Principios de Diagramación

```
1. Cada diagrama debe tener un TÍTULO descriptivo.
2. Cada diagrama debe tener una LEYENDA que explique la notación.
3. Usa formas CONSISTENTES (rectángulos, cilindros, personas).
4. Las flechas deben tener DIRECCIÓN y un ANCLA (texto descriptivo).
5. Un diagrama = UNA historia. No intentes mostrar todo en uno.
```

### Herramientas

| Herramienta | Mejor para | Formato |
|-------------|-----------|---------|
| **Structurizr DSL** | C4 como código | DSL → diagramas |
| **PlantUML** | Diagramas UML, secuencia, C4 | Texto → diagrama |
| **Mermaid** | Diagramas en Markdown/GitHub | Texto → diagrama |
| **diagrams.net (draw.io)** | Diagramas visuales rápidos | Visual |
| **Excalidraw** | Diagramas tipo pizarra | Visual, estilo informal |
| **Lucidchart** | Colaboración empresarial | Visual |
| **C4-PlantUML** | C4 con estereotipos UML | Texto → diagrama |
| **Ilograph** | Diagramas interactivos | Visual interactivo |

### C4 como Código (Structurizr DSL)

```c4
workspace "E-Commerce System" "Sistema de ventas online" {

    model {
        customer = person "Cliente" "Usuario que realiza compras"
        admin = person "Administrador" "Gestiona catálogo y pedidos"

        spa = container "SPA" "React" "Aplicación web del cliente"
        api = container "API" "Node.js + Express" "API REST principal"
        db = container "PostgreSQL" "PostgreSQL 16" "Almacena pedidos, productos, usuarios"
        cache = container "Redis" "Redis 7" "Caché de sesiones y productos"

        customer -> spa "Usa"
        admin -> spa "Usa"
        spa -> api "Realiza peticiones" "JSON/HTTPS"
        api -> db "Lee y escribe" "JDBC"
        api -> cache "Cachea" "Redis Protocol"
    }

    views {
        systemContext ecommerce "SystemContext" {
            include *
        }

        container api "APIContainer" {
            include *
        }
    }
}
```

## 30.4 ADRs por Disciplina

Una plantilla más detallada para decisiones importantes:

```markdown
# ADR-023: Adoptar PostgreSQL como Base de Datos Primaria

## Metadata
- **ADR ID**: 023
- **Fecha**: 2024-06-15
- **Estado**: Aceptado
- **Decisor**: Arquitecto Principal
- **Consultados**: Tech Leads, DBA, VP Ingeniería
- **Influenciados**: Equipos de desarrollo, SRE

## Contexto y Problema
[Describe el contexto técnico y de negocio.
¿Qué problema estamos resolviendo? ¿Qué restricciones existen?]

Seleccionar la base de datos primaria para el nuevo sistema.
Datos altamente relacionales (usuarios, pedidos, productos, pagos).
Equipo con experiencia en PostgreSQL y MongoDB.

## Supuestos
- El volumen de datos no excederá 5TB en los próximos 3 años.
- El equipo tiene presupuesto para RDS gestionado.
- No se requiere escalabilidad horizontal de escritura en el corto plazo.

## Opciones Consideradas

### Opción 1: PostgreSQL (Elegida)
- **Ventajas**: ACID, ecosistema maduro, equipo experimentado, soporte JSONB.
- **Desventajas**: Escalabilidad limitada a read replicas, no nativamente distribuido.

### Opción 2: MongoDB
- **Ventajas**: Esquema flexible, escalabilidad horizontal nativa.
- **Desventajas**: Transacciones multi-documento con limitaciones, equipo sin experiencia profunda.

### Opción 3: CockroachDB
- **Ventajas**: PostgreSQL-compatible, distribuido nativamente.
- **Desventajas**: Operativamente complejo, ecosistema de herramientas menor.

## Decisión
Adoptar PostgreSQL 16 (AWS RDS) como base de datos primaria.

**Justificación**: ACID sin concesiones + experiencia del equipo + RDS reduce carga operativa.

## Consecuencias

### Positivas
- Transacciones ACID robustas para pagos y pedidos.
- RDS: backups automáticos, point-in-time recovery, minor version upgrades.

### Negativas
- Escalabilidad limitada a read replicas. Si necesitamos sharding en el futuro,
  la migración será significativa.

### Riesgos
- **Riesgo**: Crecimiento de datos excede capacidad vertical de RDS.
- **Mitigación**: Monitoreo trimestral de crecimiento. Si tendencia supera 2TB en 18 meses,
  iniciar evaluación de estrategia de particionamiento.

## Notas de Implementación
- Pool de conexiones: PgBouncer con transaction pooling.
- Read replicas para reportes y analíticas.
- Particionamiento por fecha en tablas de logs/eventos.

## Referencias
- https://www.postgresql.org/docs/16/
- Documento de arquitectura de datos v2.3
```

## 30.5 Decisiones que DEBES Documentar

| Decisión | Formato | ¿Por qué? |
|----------|---------|----------|
| Elección de lenguaje/framework | ADR | Impacta hiring, tooling, ecosistema |
| Estilo arquitectónico | ADR | Define la estructura fundamental |
| Estrategia de branching | ADR | Afecta a todos los devs |
| Elección de base de datos | ADR | La decisión más cara de revertir |
| Estrategia de autenticación | ADR | Impacta seguridad de todo el sistema |
| Protocolo de comunicación | ADR | Define cómo se hablan los servicios |
| Estrategia de versionado de API | Estándar | Todos los equipos deben seguirla |
| Convenciones de código | Guía | Consistencia en todo el código base |
| Topología de despliegue | Diagrama C4 | Operaciones necesita saberlo |
| Modelo de dominio | Diagrama | Los devs necesitan el ubiquitous language |

## 30.6 Documentación Viva (Living Documentation)

La documentación que no se mantiene muere. Estrategias para mantenerla viva:

1. **Documentación como código**: Markdown en el repo. PRs para cambios en docs.
2. **Generación automática**: OpenAPI desde código, C4 desde Structurizr, ERDs desde migraciones.
3. **Tests de documentación**: Tests que validan que los diagramas reflejan la realidad.
4. **Revisiones periódicas**: Cada trimestre, 1 hora para actualizar ADRs y diagramas.
5. **Vinculación con el backlog**: ADRs referenciados desde historias de usuario relacionadas.

### Arquitectura como Código Validable

```python
# Test que verifica que los componentes documentados existen en el código
def test_documented_components_exist():
    """Verifica que todos los componentes del diagrama C4 existen en el código."""
    documented_components = parse_c4_diagram("docs/architecture/c4-container.puml")
    actual_packages = scan_packages("./src")

    for component in documented_components:
        assert component.name in actual_packages, \
            f"Componente documentado '{component.name}' no encontrado en el código"

def test_database_reflects_model():
    """Verifica que el esquema de BD refleja el modelo documentado."""
    documented_tables = parse_er_diagram("docs/architecture/er-model.puml")
    actual_tables = db.get_tables()

    for table in documented_tables:
        assert table in actual_tables, \
            f"Tabla documentada '{table}' no existe en la base de datos"
```

## 30.7 Anti-Patrones de Documentación

| Anti-Patrón | Por qué es Malo | Solución |
|-------------|----------------|----------|
| **Documentación-Wiki** | Se pudre, nadie la actualiza. | Docs as Code en el repo |
| **Diagrama-Frankenstein** | Un solo diagrama con 50 cajas, ilegible. | Múltiples niveles C4 |
| **Documentación-Premio** | 200 págs que nadie pidió. | Justo lo necesario (lean) |
| **Notación-Secreta** | Solo el creador entiende los símbolos. | Leyenda + notación estándar |
| **Diagrama-Sin-Fecha** | No sabes si es actual o de hace 3 años. | Fecha + versión en cada diagrama |
| **Documentación-Perfecta** | Documentar todo "por si acaso". | Documenta solo lo que duele no saber |

## 30.8 Cuándo Documentar y Cuándo NO

**SÍ documenta:**
- Decisiones que serán difíciles de revertir.
- Interfaces entre equipos (contratos de API).
- Conceptos de dominio complejos.
- Patrones arquitectónicos no obvios.
- Restricciones técnicas que no son evidentes leyendo el código.

**NO documentes:**
- Lo que el código ya expresa claramente.
- Detalles de implementación que cambian cada sprint.
- Decisiones triviales (qué librería de logging usar).
- Diagramas de clases completos (el IDE los genera).
- Documentación que no tienes intención de mantener.

> "Documenta como si el próximo arquitecto fuera un asesino psicópata que sabe dónde vives." — Atribuido a varios

---

> **Reflexión del capítulo**: La documentación de arquitectura es una inversión en la memoria institucional. El código dice "qué" y "cómo". La documentación de arquitectura debe decir "por qué". Si tu documentación compite con el código como fuente de verdad, estás haciendo algo mal. La documentación complementa al código, no lo reemplaza.

---

← [Capítulo anterior](29-entrevistas.md) | [Inicio](README.md) | [Capítulo siguiente →](31-testing.md)
