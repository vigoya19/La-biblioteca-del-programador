# Capítulo 33: Gobernanza de Arquitectura y Fitness Functions

> "No puedes controlar lo que no mides. No puedes mejorar lo que no gobiernas." — Adaptado de Peter Drucker

## 33.1 El Problema: La Entropía Arquitectónica

Todo sistema tiende al desorden. Sin gobernanza, tu hermosa arquitectura se degrada sprint a sprint:

```
Semana 1:   Clean Architecture, módulos separados. ✓
Semana 10:  "Solo un import cruzado, es urgente."
Semana 50:  "¿Quién puso lógica de negocio en el controlador REST?"
Semana 100: "Tenemos un monolito, pero sin los beneficios del monolito."
```

La gobernanza de arquitectura es el conjunto de prácticas, herramientas y procesos que aseguran que el sistema evoluciona dentro de los límites arquitectónicos definidos.

## 33.2 Fitness Functions — Gobernanza Automatizada

Propuestas por Neal Ford en *Building Evolutionary Architectures*. Son tests automatizados que validan que las características arquitectónicas se mantienen en el tiempo.

> "Una fitness function es cualquier mecanismo que proporciona una evaluación objetiva de la integridad de una característica arquitectónica."

### Ejemplos de Fitness Functions

```python
# Fitness Function 1: No acoplamiento entre módulos
def test_module_dependency_direction():
    """
    Verifica que las dependencias solo apuntan hacia adentro.
    Dominio ← Aplicación ← Infraestructura ← Adaptadores
    """
    violations = check_dependency_rules([
        DependencyRule("domain", should_not_depend_on=["application", "infrastructure", "adapter"]),
        DependencyRule("application", should_not_depend_on=["infrastructure", "adapter"]),
        DependencyRule("infrastructure", should_not_depend_on=["adapter"]),
    ])

    assert violations == [], f"Violaciones de dependencia: {violations}"


# Fitness Function 2: Nombrado de paquetes
def test_package_naming_convention():
    """Verifica que los paquetes siguen la convención."""
    packages = scan_packages("./src")

    for pkg in packages:
        assert pkg.startswith("com.shopflow."), \
            f"Paquete {pkg} no sigue la convención de naming"

        # El dominio no debe tener nombres técnicos
        if "domain" in pkg:
            assert not any(tech in pkg for tech in
                ["controller", "repository_impl", "mapper", "dto"]), \
                f"Paquete de dominio {pkg} contiene nombres de infraestructura"


# Fitness Function 3: No dependencias cíclicas
def test_no_circular_dependencies():
    """Verifica que no hay ciclos de dependencia entre módulos."""
    graph = build_dependency_graph("./src")

    cycles = graph.find_cycles()

    assert len(cycles) == 0, \
        f"Dependencias cíclicas detectadas: {cycles}"


# Fitness Function 4: Tamaño máximo de clase
def test_max_class_size():
    """Verifica que ninguna clase excede 300 líneas."""
    max_lines = 300
    violations = []

    for file in scan_java_files("./src"):
        lines = count_lines(file)
        if lines > max_lines:
            violations.append(f"{file}: {lines} líneas")

    assert len(violations) == 0, \
        f"Clases que exceden {max_lines} líneas:\n" + "\n".join(violations)


# Fitness Function 5: Sin SQL en capa de aplicación
def test_no_sql_in_application_layer():
    """Verifica que la capa de aplicación no contiene SQL."""
    files = scan_files("./src/application", [".java", ".kt"])
    sql_patterns = ["SELECT ", "INSERT ", "UPDATE ", "DELETE ", "@Query"]

    violations = []
    for file in files:
        content = read_file(file)
        for pattern in sql_patterns:
            if pattern in content:
                violations.append(f"{file}: contiene '{pattern}'")

    assert len(violations) == 0, \
        f"SQL encontrado en capa de aplicación:\n" + "\n".join(violations)


# Fitness Function 6: Atributo de calidad — Latencia
def test_p95_latency_under_slo():
    """
    Verifica que el p95 de latencia está bajo el SLO.
    Se ejecuta contra staging con datos realistas.
    """
    load_test = k6.run("load-tests/latency-test.js")
    p95 = load_test.metrics.http_req_duration.p95

    assert p95 < 200, \
        f"p95 latency ({p95}ms) excede SLO (200ms)"
```

### Fitness Functions en el Pipeline

```
┌────────────────────────────────────────────────────┐
│                  CI/CD Pipeline                     │
│                                                     │
│  ┌──────┐    ┌──────┐    ┌─────────────┐           │
│  │Compil│───►│ Test │───►│Fitness Funcs │───► Deploy│
│  │      │    │ Unit │    │              │           │
│  └──────┘    └──────┘    └─────────────┘           │
│                           │                         │
│                           ▼                         │
│                    ¿Pasa todas las                  │
│                    fitness functions?               │
│                     ├── SÍ → Normal                 │
│                     └── NO → Bloquear PR            │
│                              Notificar arquitecto   │
└────────────────────────────────────────────────────┘
```

## 33.3 Herramientas de Gobernanza

### ArchUnit (Java)
```java
// Verificar reglas de arquitectura en tests
@Test
void domain_should_not_depend_on_infrastructure() {
    classes()
        .that().resideInAPackage("..domain..")
        .should().onlyDependOnClassesThat()
        .resideInAnyPackage("..domain..", "java..", "org.slf4j..")
        .check(classes);
}

@Test
void controllers_should_be_annotated_with_RestController() {
    classes()
        .that().haveSimpleNameEndingWith("Controller")
        .should().beAnnotatedWith(RestController.class)
        .check(classes);
}

@Test
void repositories_should_be_interfaces() {
    classes()
        .that().haveSimpleNameEndingWith("Repository")
        .should().beInterfaces()
        .check(classes);
}
```

### NetArchTest (.NET)
```csharp
// Reglas de arquitectura programáticas
var result = Types.InCurrentDomain()
    .That().ResideInNamespace("Domain")
    .Should().NotHaveDependencyOn("Infrastructure")
    .GetResult();

Assert.True(result.IsSuccessful);
```

### Dependency Cruiser (JavaScript/TypeScript)
```javascript
// .dependency-cruiser.js
module.exports = {
  forbidden: [
    {
      name: 'domain-should-not-depend-on-infrastructure',
      from: { path: '^src/domain' },
      to: { path: '^src/infrastructure' },
    },
    {
      name: 'no-circular-dependencies',
      from: {},
      to: { circular: true },
    },
  ],
};
```

### Moduliths (Spring Modulith)
```java
// Verificar modularidad en Spring Boot
@SpringBootTest
class ModularityTest {
    @Test
    void verifyModularity() {
        ApplicationModules.of(ShopFlowApplication.class)
            .verify(); // Lanza error si hay violaciones
    }
}
```

## 33.4 Modelos de Gobernanza

### Gobernanza Centralizada
```
┌─────────────────────┐
│ Architecture Board  │  ← Decide TODO
│ (Arquitecto Senior) │
└──────────┬──────────┘
           │
    ┌──────┼──────┐
    ▼      ▼      ▼
Equipo A Equipo B Equipo C
```

**Cuándo**: Organizaciones pequeñas, sistemas críticos (financiero, salud).
**Riesgo**: Cuello de botella, arquitecto de marfil.

### Gobernanza Federada (Recomendado)
```
┌──────────────────────────────────┐
│ Architecture Guild (comunidad)   │
│ Define principios, no detalles   │
└──────────┬───────────────────────┘
           │
    ┌──────┼──────┐
    ▼      ▼      ▼
Equipo A Equipo B Equipo C
(autonomía dentro de principios)
```

**Cuándo**: Organizaciones medianas-grandes, equipos maduros.
**Beneficio**: Autonomía con alineación.

### Gobernanza por Plataforma
```
Internal Developer Platform:
  - Golden path templates (estándares embebidos en herramientas)
  - Fitness functions automáticas en CI/CD
  - "Es más fácil hacerlo bien que hacerlo mal"

La gobernanza no es un comité, es una plataforma.
```

## 33.5 Principios Arquitectónicos vs Reglas

| Tipo | Ejemplo | Enforcement |
|------|---------|-------------|
| **Principio** | "Prefiere SOLID" | Manual (code review) |
| **Estándar** | "Usa Postgres 16 para BD primarias" | Fitness function + guía |
| **Regla** | "El dominio no importa infraestructura" | Bloqueo automático en CI |
| **Convención** | "Nombra paquetes en singular" | Linter/formatter |

**No todo debe ser regla. No todo puede ser sugerencia.** Jerarquiza:

```
1. Reglas de seguridad: SIEMPRE obligatorias, bloquean el pipeline.
2. Reglas de arquitectura: Obligatorias, bloquean PRs.
3. Estándares: Fuertemente recomendados, PRs con justificación si se desvían.
4. Convenciones: Automatizables con linter/formatter.
5. Principios: Guían, no bloquean. Cultura > enforcement.
```

## 33.6 El Architecture Review

Proceso formal (o informal) para evaluar decisiones arquitectónicas.

### Tipos de Reviews

| Tipo | Frecuencia | Participantes | Objetivo |
|------|-----------|---------------|----------|
| **Lightweight** | Semanal/bi-semanal | Arquitecto + Tech Lead | Revisar decisiones tácticas |
| **Design Review** | Por feature compleja | Arquitecto + Equipo | Validar diseño antes de implementar |
| **Architecture Review** | Trimestral | Arquitectos + Principals | Evaluar salud de la arquitectura |
| **Strategic Review** | Anual/Semestral | CTO + Architects | Alinear arquitectura con estrategia |

### Template de Architecture Review

```
## Architecture Review: [Sistema/Módulo]
Fecha: [fecha]
Revisores: [nombres]
Presenta: [equipo]

### Contexto
[Qué problema resuelve, qué decisiones se tomaron]

### Decisiones Clave
1. [Decisión 1 — ADR-XXX]
2. [Decisión 2 — ADR-XXX]

### Verificación de Atributos de Calidad
[ ] Rendimiento: ¿Cumple SLOs? Evidencia:
[ ] Seguridad: ¿Threat model completado? Evidencia:
[ ] Escalabilidad: ¿Load test superado? Evidencia:
[ ] Mantenibilidad: ¿Fitness functions pasan? Evidencia:

### Riesgos Identificados
1. [Riesgo] — Mitigación: [plan]
2. [Riesgo] — Mitigación: [plan]

### Deuda Técnica Acumulada
[Listar y priorizar]

### Acciones
1. [Acción] — Responsable: [nombre] — Fecha: [fecha]
2. [Acción] — Responsable: [nombre] — Fecha: [fecha]

### Veredicto
[ ] Aprobado sin condiciones
[ ] Aprobado con recomendaciones
[ ] Requiere cambios (re-review en X semanas)
[ ] Rechazado (rediseñar)
```

## 33.7 El Arquitecto como Guardián (y Cuándo Soltar)

### Lo Que Sí Debes Gobernar
- Estilo arquitectónico (monolito vs microservicios).
- Elección de bases de datos y message brokers.
- Protocolos de comunicación entre servicios.
- Estrategia de autenticación y autorización.
- Contratos de API públicamente expuestos.
- Reglas de dependencia entre módulos principales.

### Lo Que NO Debes Gobernar
- Qué librería de logging usa cada equipo.
- Cómo nombran sus variables (el linter lo maneja).
- Qué framework CSS usa el frontend.
- Detalles de implementación interna.
- Cómo organizan sus archivos dentro del módulo.

> "Gobierna las interfaces entre equipos. El interior de cada módulo es sagrado."

## 33.8 Métricas de Gobernanza

¿Cómo sabes si tu gobernanza está funcionando?

| Métrica | Señal de Alarma |
|---------|----------------|
| **Tiempo promedio de Architecture Review** | >2 días → cuello de botella |
| **Fitness function pass rate** | <95% → arquitectura degradándose |
| **ADR velocity** | 0 ADRs en 3 meses → decisiones sin documentar |
| **Desviaciones de estándar (con justificación)** | Demasiadas → estándar incorrecto |
| **Violaciones de dependencia (por sprint)** | Tendencia creciente → alerta roja |

---

> **Reflexión del capítulo**: La gobernanza efectiva no se siente como burocracia. Se siente como un guardarraíl en la carretera: no te impide conducir, pero te salva cuando te desvías. Invierte en fitness functions automáticas. La mejor gobernanza es la que ocurre sin que nadie se dé cuenta.
