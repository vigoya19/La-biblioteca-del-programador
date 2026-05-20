# Capítulo 8: Estilos Arquitectónicos

> "No hay balas de plata. Cada estilo brilla en un contexto y falla en otro."

## 8.1 ¿Qué es un Estilo Arquitectónico?

Un estilo arquitectónico es un conjunto de principios, patrones y restricciones que definen la estructura, comunicación y comportamiento de un sistema. Es una **plantilla de alto nivel** para organizar componentes.

## 8.2 Monolithic Architecture

Toda la funcionalidad en un único deployable. No confundir con "monolito mal diseñado": un monolito bien hecho es modular y mantenible.

```
┌─────────────────────────────────────┐
│          Monolito (un solo .jar)     │
│                                      │
│  ┌──────────┐  ┌──────────────────┐ │
│  │ Módulo   │  │ Módulo           │ │
│  │ Usuarios │  │ Pedidos          │ │
│  └──────────┘  └──────────────────┘ │
│                                      │
│  ┌──────────┐  ┌──────────────────┐ │
│  │ Módulo   │  │ Módulo           │ │
│  │ Pagos    │  │ Inventario       │ │
│  └──────────┘  └──────────────────┘ │
│                                      │
│     Compartición de memoria          │
│     Llamadas en proceso              │
│     Una sola BD                      │
└─────────────────────────────────────┘
```

### Ventajas
- **Desarrollo simple**: Un repo, un build, un despliegue.
- **Debugging fácil**: Todo corre en un proceso.
- **Transacciones triviales**: ACID con una sola BD.
- **Menor latencia**: Llamadas en memoria, no en red.

### Desventajas
- **Escalabilidad limitada**: Escala todo o nada.
- **Acoplamiento**: Los módulos se contaminan con el tiempo.
- **Despliegues lentos**: Un cambio mínimo = redesplegar todo.
- **Vendor lock-in tecnológico**: Todo en el mismo stack.

### Cuándo Usarlo
- Startups y MVPs (velocidad > escalabilidad).
- Equipos pequeños (menos de 10-15 devs).
- Dominios simples que no justifican distribución.

## 8.3 Microservices Architecture

Sistema distribuido donde cada servicio es independiente: tiene su propia base de datos, su propio ciclo de despliegue, su propio equipo.

```
┌──────────┐    ┌──────────┐    ┌──────────┐
│Servicio  │    │Servicio  │    │Servicio  │
│Usuarios  │◄──►│Pedidos   │◄──►│Pagos     │
│          │    │          │    │          │
│ ┌──────┐ │    │ ┌──────┐ │    │ ┌──────┐ │
│ │  BD  │ │    │ │  BD  │ │    │ │  BD  │ │
│ └──────┘ │    │ └──────┘ │    │ └──────┘ │
└──────────┘    └──────────┘    └──────────┘
```

### Características
- **Database per Service**: Cada servicio dueño de sus datos.
- **Comunicación por red**: REST, gRPC, mensajería.
- **Despliegue independiente**: Cada servicio se despliega cuando quiere.
- **Escalabilidad granular**: Escalas solo el servicio que necesita.

### Ventajas
- **Escalabilidad**: Escala servicios individualmente.
- **Despliegues rápidos y aislados**: Un servicio = un cambio.
- **Libertad tecnológica**: Cada servicio usa el stack óptimo.
- **Resiliencia**: Un fallo no tumba todo el sistema.

### Desventajas
- **Complejidad operativa**: Red, latencia, fallos parciales, distributed tracing.
- **Transacciones distribuidas**: No hay ACID fácil; necesitas Sagas.
- **Consistencia eventual**: Los datos se desincronizan temporalmente.
- **Debugging difícil**: Un request cruza 5 servicios.
- **Sobrecarga de red**: Latencia acumulada.

### Cuándo Usarlo
- Organizaciones con múltiples equipos autónomos.
- Sistemas que necesitan escalar partes específicas.
- Cuando diferentes módulos necesitan stacks diferentes.

### Cuándo NO Usarlo
- Startups en etapa temprana (empieza con monolito modular).
- Equipos pequeños (la complejidad mata la productividad).
- Dominios simples que no justifican distribución.

## 8.4 Modular Monolith

El punto dulce entre monolito y microservicios: módulos bien separados lógicamente, pero desplegados juntos.

```
┌───────────────────────────────────────┐
│         Modular Monolith              │
│                                       │
│  ┌─────────┐ ←solo API→ ┌──────────┐ │
│  │ Módulo  │            │ Módulo   │ │
│  │ Usuarios│            │ Pedidos  │ │
│  └─────────┘            └──────────┘ │
│       │                      │       │
│       ▼                      ▼       │
│  ┌─────────┐            ┌──────────┐ │
│  │ BD      │            │ BD       │ │
│  │ Usuarios│            │ Pedidos  │ │
│  └─────────┘            └──────────┘ │
│                                       │
│  Separación lógica + física en BD     │
│  Despliegue único                     │
└───────────────────────────────────────┘
```

**Estrategia de migración**: Monolito → Monolito Modular → Microservicios (solo si es necesario).

## 8.5 Event-Driven Architecture

Los componentes se comunican produciendo y consumiendo eventos. Desacoplamiento temporal y espacial.

```
Productor A ──► ┌──────────────┐ ──► Consumidor X
                │              │
Productor B ──► │ Event Broker │ ──► Consumidor Y
                │  (Kafka,     │
Productor C ──► │   RabbitMQ)  │ ──► Consumidor Z
                └──────────────┘
```

### Patrones Clave
- **Event Notification**: "Algo pasó" (PedidoCreado).
- **Event-Carried State Transfer**: El evento contiene todos los datos necesarios.
- **Event Sourcing**: El estado se reconstruye de una secuencia de eventos.
- **CQRS**: Separar lecturas de escrituras (ver Capítulo 11).

## 8.6 Service-Oriented Architecture (SOA)

Predecesor de microservicios. Servicios reutilizables comunicados vía Enterprise Service Bus (ESB).

**Diferencias con Microservicios**:
- SOA comparte BD; microservicios no.
- SOA usa protocolos pesados (SOAP/XML); microservicios prefieren REST/gRPC.
- SOA busca reutilización; microservicios priorizan autonomía.
- SOA centraliza con ESB; microservicios son "smart endpoints, dumb pipes".

## 8.7 Comparativa Rápida

| Estilo | Escalabilidad | Complejidad | Acoplamiento | Uso Típico |
|--------|--------------|-------------|-------------|------------|
| **Monolito** | Baja | Baja | Alto | MVPs, equipos pequeños |
| **Monolito Modular** | Media | Media | Medio | Crecimiento controlado |
| **Microservicios** | Alta | Alta | Bajo | Grandes organizaciones |
| **Event-Driven** | Alta | Muy Alta | Muy Bajo | Alta asincronía |
| **SOA** | Media | Alta | Medio | Empresas legacy |

## 8.8 Cómo Elegir

1. **Empieza con monolito modular** hasta que duela.
2. **Extrae microservicios** solo en los puntos de dolor reales (escalabilidad, autonomía de equipos, despliegues).
3. **Adopta event-driven** para desacoplar flujos asíncronos naturales (notificaciones, integración con terceros).
4. **No uses SOA** en sistemas nuevos; es un legado.

---

> **Reflexión del capítulo**: No existen decisiones arquitectónicas "correctas" en abstracto. Todas dependen del contexto. La madurez del arquitecto se mide por su capacidad de elegir el estilo adecuado para las restricciones reales, no por seguir tendencias.
