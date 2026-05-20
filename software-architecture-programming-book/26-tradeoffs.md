# Capítulo 26: Estrategia, Trade-offs y Toma de Decisiones

> "No existe la mejor arquitectura. Solo existen arquitecturas que maximizan ciertos atributos a costa de otros."

## 26.1 La Naturaleza de los Trade-offs

Cada decisión arquitectónica favorece unos atributos de calidad y perjudica otros. No existe la decisión "correcta", existe la decisión "correcta para tu contexto".

```
Ejemplo: Elegir entre Monolito y Microservicios

Monolito:
  ✓ Simplicidad de desarrollo
  ✓ Performance (sin latencia de red)
  ✓ Transacciones ACID simples
  ✗ Escalabilidad limitada
  ✗ Acoplamiento de equipos

Microservicios:
  ✓ Escalabilidad granular
  ✓ Autonomía de equipos
  ✓ Despliegues independientes
  ✗ Complejidad operativa
  ✗ Latencia de red
  ✗ Consistencia eventual
```

**La decisión correcta depende de**: tamaño del equipo, presupuesto, time-to-market, complejidad del dominio, expectativas de crecimiento.

## 26.2 Architecture Decision Records (ADRs)

Documentar decisiones arquitectónicas de forma ligera y estructurada. No necesitas un documento de 50 páginas.

### Plantilla ADR

```markdown
# ADR-004: Usar PostgreSQL como base de datos primaria

## Estado
Aceptado (2024-01-15)

## Contexto
Necesitamos elegir una base de datos para el nuevo sistema de pedidos.
Los datos son altamente relacionales (clientes, pedidos, items, pagos).
Necesitamos transacciones ACID y consultas complejas.

## Decisión
Usaremos PostgreSQL 16 como base de datos primaria.

## Alternativas Consideradas
1. MongoDB: Descartada por falta de transacciones multi-documento robustas.
2. MySQL: Descartada por menor soporte de features avanzadas (JSONB, GIN indexes).
3. CockroachDB: Interesante para escalabilidad horizontal, pero añade complejidad operativa.

## Consecuencias
- Positivo: ACID, ecosistema maduro, equipo tiene experiencia.
- Negativo: Escalabilidad vertical limitada (mitigación: read replicas).
- Neutro: Vendor lock-in moderado.
```

### Cuándo Escribir un ADR
- Decisiones que afectan la estructura del sistema.
- Elecciones de tecnología significativas.
- Cambios en la estrategia de arquitectura.
- No para decisiones triviales (qué librería de logging usar).

## 26.3 Método de Evaluación Tecnológica

Cuando evalúas una tecnología, considera:

### Radar de Tecnología (ThoughtWorks)
```
Hold        → No usar. Problemas conocidos, evitar.
Assess      → Investigar. Prometedor pero inmaduro.
Trial       → Probar en proyecto no crítico.
Adopt       → Usar. Probado, maduro, recomendado.
```

### Matriz de Decisión Ponderada

| Criterio | Peso | PostgreSQL | MongoDB | DynamoDB |
|----------|------|-----------|---------|----------|
| Experiencia equipo | 25% | 4 | 2 | 1 |
| Transacciones ACID | 30% | 5 | 3 | 2 |
| Escalabilidad | 20% | 3 | 4 | 5 |
| Costo operativo | 15% | 4 | 3 | 5 |
| Ecosistema/herramientas | 10% | 5 | 4 | 3 |
| **Puntuación** | | **4.2** | **3.05** | **2.9** |

No es una fórmula mágica, pero estructura la discusión y evita decisiones por "moda".

## 26.4 Estrategia de Migración Arquitectónica

Cambiar una arquitectura en producción sin romper todo requiere estrategia.

### Strangler Fig Pattern
Envolver el sistema legacy con nuevas capacidades hasta que el legacy se atrofia y muere.

```
For each feature in legacy:
  1. Proxy request to legacy
  2. Build new implementation
  3. Route traffic to new (canary: 5% → 50% → 100%)
  4. Deprecate legacy route

     ┌──────────────┐
     │   Router     │
     │ (Feature Flag)│
     └──┬────────┬──┘
  100%  │        │  0% (inicialmente)
   ┌────▼────┐ ┌─▼──────────┐
   │ Legacy  │ │ New Service │
   │Monolith │ │             │
   └─────────┘ └────────────┘

  Con el tiempo: 0% legacy, 100% nuevo
```

### Branch by Abstraction
En lugar de feature branches largos:
1. Crear abstracción (interfaz) del componente a cambiar.
2. Implementar la nueva versión tras la abstracción.
3. Cambiar entre versiones vía configuración.
4. Eliminar la versión antigua.

## 26.5 Gestión de Deuda Técnica

### Tipos de Deuda Técnica

| Tipo | Ejemplo | Estrategia |
|------|---------|-----------|
| **Deliberada** | "Lo hacemos rápido, ya lo arreglamos" | Registrar, planificar, pagar pronto |
| **Accidental** | Mal diseño por inexperiencia | Refactor gradual |
| **Bit rot** | Dependencias desactualizadas | Renovación periódica (cada sprint) |
| **Estratégica** | Microservicios para 10 usuarios | Aceptar y monitorear |

### Cómo Manejarla
1. **Hazla visible**: Etiqueta en backlog, comentarios en PR.
2. **Boy Scout Rule**: Siempre deja el código mejor de lo que lo encontraste.
3. **Dedica tiempo**: 10-20% de cada sprint a mejora técnica.
4. **Mide el impacto**: ¿Cuánto tiempo perdemos por esta deuda?
5. **Prioriza**: Arregla lo que más duele, no lo que más molesta.

## 26.6 Ética del Arquitecto

Decisiones que tomas tienen impacto en personas reales:

- **Privacidad**: ¿Estás recolectando más datos de los necesarios?
- **Accesibilidad**: ¿Tu sistema puede ser usado por personas con discapacidades?
- **Sesgo algorítmico**: ¿Tus decisiones de ML discriminan involuntariamente?
- **Sostenibilidad**: ¿Tu arquitectura es eficiente energéticamente?
- **Vendor lock-in**: ¿Estás atrapando a tu cliente en tu ecosistema?

---

> **Reflexión del capítulo**: La maestría en arquitectura no se demuestra eligiendo la tecnología más nueva, sino eligiendo la adecuada para el contexto y documentando por qué. Un ADR bien escrito vale más que 100 diagramas. Una decisión consciente, aunque imperfecta, es mejor que una decisión por omisión.

---

← [Capítulo anterior](25-observabilidad.md) | [Inicio](README.md) | [Capítulo siguiente →](27-futuro.md)
