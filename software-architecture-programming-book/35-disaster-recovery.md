# Capítulo 35: Disaster Recovery y Continuidad de Negocio

> "La pregunta no es si tu sistema fallará catastróficamente. La pregunta es si sobrevivirás cuando pase."

## 35.1 Entendiendo de Qué Hablamos

### Las Diferencias que Importan

```
┌─────────────────────────────────────────────────────────────┐
│ CONCEPTOS QUE TODO ARQUITECTO DEBE DISTINGUIR               │
│                                                              │
│ Alta Disponibilidad (HA):                                    │
│   El sistema sigue funcionando cuando un componente falla.  │
│   Ej: Un servidor muere, el tráfico va a otro.              │
│   Tiempo: Segundos. No hay intervención humana.             │
│                                                              │
│ Disaster Recovery (DR):                                      │
│   Recuperas el sistema después de un desastre.              │
│   Ej: La región cloud completa está caída.                  │
│   Tiempo: Minutos, horas o días. Requiere intervención.     │
│                                                              │
│ Continuidad de Negocio (BC):                                 │
│   El negocio sigue operando aunque el sistema falle.         │
│   Ej: Procesar pedidos en papel mientras el sistema vuelve. │
│   Esto no es tecnología, es proceso de negocio.              │
└─────────────────────────────────────────────────────────────┘
```

### Por Qué Necesitas Entender Esto Como Arquitecto

Un desarrollador piensa en que su código no tenga bugs. Un arquitecto piensa en qué pasa cuando TODO falla simultáneamente.

```
Escenario real que me tocó vivir:

3:17 AM. El datacenter principal en Virginia sufre un incendio.
Temperatura, humo, fire suppression system activado.
TODO apagado: servidores, storage, red.

Preguntas que el CEO me hizo a las 3:45 AM:
1. "¿Cuánto tardamos en estar online?"
2. "¿Perdimos datos?"
3. "¿Los clientes van a notar algo?"

El arquitecto que no tiene respuestas para esto a las 3 AM
no está haciendo su trabajo.
```

## 35.2 Las Dos Métricas que Definen Todo

### RPO (Recovery Point Objective) — ¿Cuántos datos estoy dispuesto a perder?

```
RPO = Máxima cantidad de datos que aceptas perder en un desastre,
     medida en tiempo.

Ejemplos:

RPO = 0         → No perder NADA. Cada transacción replicada
                  sincrónicamente. Cuesta caro.
RPO = 1 minuto  → Puedo perder el último minuto de datos.
RPO = 1 hora    → Puedo perder la última hora.
RPO = 24 horas  → Respaldo una vez al día.
```

```
Visual:
─────┬──────────────┬──────────────┬──────────────┬─────► tiempo
     │              │              │              │
  Último         FAILURE         RPO             Ahora
  backup
     └───────────────────────────────────────────┘
                  Datos potencialmente perdidos
```

### RTO (Recovery Time Objective) — ¿Cuánto tiempo puedo estar caído?

```
RTO = Tiempo máximo aceptable para restaurar el servicio.

Ejemplos:

RTO = 0          → Failover instantáneo. Multi-AZ activo-activo. $$$$
RTO = 5 minutos  → DNS failover a otra región. Automático.
RTO = 1 hora     → Levantar infraestructura de respaldo. Semi-automático.
RTO = 4 horas    → Restaurar backups, configurar, validar. Manual.
RTO = 24 horas   → Recibir hardware nuevo, reconfigurar todo. Barato.
```

```
Visual:
─────┬─────────────────────────────────────┬─────► tiempo
     │                                     │
  FAILURE                              Recuperado
     └─────────────────────────────────────┘
                    RTO (downtime)
```

### La Relación entre RPO y RTO

```
Puedes tener RPO bajo y RTO alto:
  → Haces backup cada minuto (RPO=1min), pero tardas 4 horas en restaurar (RTO=4h).
  
Puedes tener RTO bajo y RPO alto:
  → Failover instantáneo a DR (RTO=0), pero el último backup es de ayer (RPO=24h).

Lo ideal: ambos bajos. Pero $$$.
```

## 35.3 Cómo Definir RPO y RTO para tu Sistema

### No es una Decisión Técnica, es de Negocio

```
No preguntes: "¿Qué RPO/RTO podemos lograr con PostgreSQL?"
Pregunta:     "¿Cuánto cuesta una hora de downtime?"
```

### Ejercicio de Cálculo

```
Sistema de E-commerce:

Ingresos por hora: $50,000
Pérdida de reputación por hora caído: Difícil de cuantificar
                                      (clientes que no vuelven)
Costo de cumplir RTO=1min: $15,000/mes
Costo de cumplir RTO=1hora: $2,000/mes

Cálculo:
  RTO=1 hora → 1 downtime de 1 hora = $50,000 perdidos + reputación
  RTO=1 min  → 1 downtime de 1 min   = $833 perdidos

Decisión del negocio:
  "Aceptamos perder hasta 1 hora. El costo de infraestructura
   para 1 minuto no se justifica. RTO=1 hora."

ESTA conversación la debe liderar el arquitecto con el CFO/CEO.
No se decide en un sprint planning.
```

### Tabla de Referencia por Industria

| Industria | RPO Típico | RTO Típico | Por Qué |
|-----------|-----------|-----------|---------|
| **Financiera (trading)** | 0 (síncrono) | <1 min | Cada milisegundo caído pierde millones |
| **Financiera (banca)** | Segundos | <1 hora | Regulaciones, aunque no trading |
| **E-commerce** | <1 hora | <4 horas | Cada hora caída = ventas perdidas |
| **Salud (emergencias)** | 0 | <5 min | Vidas dependen del sistema |
| **SaaS B2B** | <1 hora | <4 horas | Clientes esperan, pero no es vida/muerte |
| **Contenido/Blogs** | 24 horas | 24 horas | El contenido no es crítico |
| **Startup MVP** | 24 horas | "Cuando podamos" | Prioridad: features, no DR |

## 35.4 Estrategias de Disaster Recovery

### Estrategia 1: Backup & Restore (La Más Simple)

```
┌──────────────────────┐     ┌──────────────────────┐
│  Región Primaria     │     │  Región DR            │
│  (us-east-1)         │     │  (us-west-2)          │
│                       │     │                       │
│  ┌──────┐ ┌────────┐ │     │  (infraestructura     │
│  │  BD  │ │  S3    │ │     │   NO desplegada,      │
│  └──┬───┘ └───┬────┘ │     │   se crea bajo demanda)│
│     │         │       │     │                       │
│     ▼         ▼       │     │  ┌──────────────────┐ │
│  Backup ──────┼───────┼─────┼─► S3 (cross-region) │ │
│  cada 1h      │       │     │  └──────────────────┘ │
│               └───────┼─────┼─► S3 (cross-region) │ │
│                       │     │                       │
└──────────────────────┘     └──────────────────────┘

Procedimiento de Recuperación:
  1. Detectar desastre en primaria.
  2. Terraform apply en región DR (levanta infraestructura).
  3. Restaurar BD desde último backup en S3.
  4. Apuntar DNS a nueva región.
  5. Validar que funciona.

RPO: 1 hora (frecuencia del backup)
RTO: 1-4 horas (Terraform + restore + validar)
Costo: BAJO (solo pagas storage entre desastres)
```

### Estrategia 2: Pilot Light (Luz Piloto)

```
┌──────────────────────┐     ┌──────────────────────┐
│  Región Primaria     │     │  Región DR            │
│                       │     │                       │
│  ┌──────┐ ┌────────┐ │     │  ┌──────┐ ┌────────┐ │
│  │  BD  │ │  App   │ │     │  │  BD  │ │ (App   │ │
│  │activa│ │5 inst. │ │     │  │réplica│ │ apagada)│ │
│  └──┬───┘ └───┬────┘ │     │  └──┬───┘ └────────┘ │
│     │         │       │     │     │                 │
│     │    ┌────▼────┐  │     │     │                 │
│     └───►│Replica- │──┼─────┼─────┘                 │
│          │ción cont│  │     │  (replicación continua)│
│          └─────────┘  │     │                       │
└──────────────────────┘     └──────────────────────┘

Procedimiento:
  1. Detectar desastre.
  2. Encender app en DR (ya existe, solo apagar réplica y
     promoverla a primaria, escalar de 0 a 5 instancias).
  3. Apuntar DNS a DR.
  4. Validar.

RPO: Segundos (replicación continua)
RTO: 15-60 minutos (encender + validar)
Costo: MEDIO (BD réplica siempre corriendo + app apagada)
```

### Estrategia 3: Warm Standby (Respaldo Caliente)

```
┌──────────────────────┐     ┌──────────────────────┐
│  Región Primaria     │     │  Región DR            │
│                       │     │                       │
│  ┌──────┐ ┌────────┐ │     │  ┌──────┐ ┌────────┐ │
│  │  BD  │ │  App   │ │     │  │  BD  │ │  App   │ │
│  │activa│ │5 inst. │ │     │  │réplica│ │1 inst. │ │
│  └──┬───┘ └───┬────┘ │     │  └──┬───┘ └────────┘ │
│     │         │       │     │     │                 │
│     └───┬─────┘       │     │     └─ ya corriendo ─┘
│         │ replicación │     │       (solo escalar)
│         └─────────────┼─────┼────────►              │
└──────────────────────┘     └──────────────────────┘

Procedimiento:
  1. Detectar desastre.
  2. Escalar app en DR de 1 a 5 instancias (ya existe).
  3. Promover BD réplica a primaria.
  4. DNS failover.

RPO: Segundos
RTO: 5-15 minutos
Costo: MEDIO-ALTO (infraestructura reducida pero siempre corriendo)
```

### Estrategia 4: Multi-Site Active-Active

```
┌──────────────────────┐     ┌──────────────────────┐
│  Región Primaria     │     │  Región DR            │
│                       │     │                       │
│  ┌──────┐ ┌────────┐ │     │  ┌──────┐ ┌────────┐ │
│  │  BD  │ │  App   │ │     │  │  BD  │ │  App   │ │
│  │activa│ │5 inst. │ │     │  │activa│ │5 inst. │ │
│  └──┬───┘ └───┬────┘ │     │  └──┬───┘ └───┬────┘ │
│     │         │       │     │     │         │       │
│     └───┬─────┘       │     │     └───┬─────┘       │
│         │ síncrono    │     │         │             │
│         └─────────────┼─────┼─────────┘             │
│                       │     │                       │
└──────────────────────┘     └──────────────────────┘
            ▲                          ▲
            │        ┌────────┐        │
            └────────┤  DNS   ├────────┘
                     │ Route53│
                     │ (ambas │
                     │ activas)│
                     └────────┘

Procedimiento:
  1. DNS detecta región caída (health check falla).
  2. Automáticamente todo el tráfico va a la región sana.
  3. Cero intervención humana (en teoría).

RPO: CERO (replicación síncrona)
RTO: CERO (failover automático)
Costo: ALTÍSIMO (infraestructura duplicada + replicación síncrona)
```

### Tabla Comparativa

| Estrategia | RPO | RTO | Costo Relativo | Para Quién |
|-----------|-----|-----|----------------|-----------|
| Backup & Restore | Horas | Horas | 1x (base) | Startups, no-críticos |
| Pilot Light | Minutos | Minutos-hora | 1.5x | Crecimiento, SaaS B2B |
| Warm Standby | Segundos | Minutos | 2x-3x | E-commerce, SaaS B2C |
| Active-Active | 0 | 0 | 4x-8x | Financiero, emergencias |

## 35.5 Disaster Recovery en la Nube (AWS Ejemplo)

### Arquitectura Multi-Región Típica

```
┌─────────────────────────────────────────────────────────────┐
│ Región Primaria: us-east-1                                   │
│                                                              │
│ Route 53 (DNS) ──► CloudFront (CDN global)                  │
│                      │                                       │
│                   ┌──▼──────────────────────────┐           │
│                   │ Multi-AZ (ya es HA dentro    │           │
│                   │ de la región: 3 AZs)         │           │
│                   │                              │           │
│                   │ ┌──────────┐ ┌────────────┐ │           │
│                   │ │   ECS    │ │ RDS (Multi │ │           │
│                   │ │ Services │ │ AZ, primaria│ │           │
│                   │ └──────────┘ └─────┬──────┘ │           │
│                   │                    │         │           │
│                   │              ┌─────▼──────┐ │           │
│                   │              │Read Replica │ │           │
│                   │              │(cross-region│─┼───────┐   │
│                   │              │ a us-west-2)│ │       │   │
│                   │              └────────────┘ │       │   │
│                   └──────────────────────────────┘       │   │
│                                                           │   │
└───────────────────────────────────────────────────────────┘   │
                                                                 │
┌───────────────────────────────────────────────────────────────┘
│
▼
┌──────────────────────────────────────────────────────────────┐
│ Región DR: us-west-2                                          │
│                                                                │
│ Read Replica (de us-east-1) ──► Se promueve a primaria        │
│                                 en desastre                   │
│                                                                │
│ ECS Services: 1 instancia corriendo, auto-scale al activar    │
│ Data Sync: S3 cross-region replication, DynamoDB Global Tables│
│                                                                │
│ Health Check: Route 53 revisa endpoint /health cada 30s       │
│ Failover: Si falla 3 veces consecutivas → activar DR          │
└──────────────────────────────────────────────────────────────┘
```

### Terraform para DR

```hcl
# Variable que controla si DR está activo
variable "dr_active" {
  description = "Set to true when primary region fails"
  type        = bool
  default     = false
}

# En DR: app corre con pocas instancias normalmente
resource "aws_ecs_service" "api_dr" {
  name            = "api-dr"
  cluster         = aws_ecs_cluster.dr.id
  task_definition = aws_ecs_task_definition.api.arn

  # Warm standby: 1 instancia normalmente
  # En desastre: cambiar a 5
  desired_count = var.dr_active ? 5 : 1

  deployment_minimum_healthy_percent = 100
  deployment_maximum_percent         = 200
}

# Código para promover DR a primaria (script que ejecutas en emergencia)
# promote_dr.sh:
# terraform apply -var="dr_active=true"
# aws rds promote-read-replica --db-instance-identifier shopflow-dr
```

## 35.6 El Plan de Disaster Recovery (Documento)

Todo lo anterior es teoría. Esto es lo que salva tu empleo a las 3 AM:

### Template del DR Plan

```markdown
# Disaster Recovery Plan — E-Commerce System
Versión: 2.1 | Última actualización: 2024-06-15
Último test: 2024-06-10 (exitosa, RTO=23min, RPO=45seg)

## 1. Información de Contacto
- Arquitecto On-Call: +1-555-0101 (rotación semanal)
- VP Ingeniería: +1-555-0102
- CTO: +1-555-0103
- AWS Support (Enterprise): case priority auto-escalation

## 2. Escenarios de Desastre
### Escenario A: Caída de región AWS completa
- Probabilidad: Baja (<0.1% anual)
- Impacto: Crítico (todo caído)
- Respuesta: Failover a DR (us-west-2)

### Escenario B: Corrupción de base de datos
- Probabilidad: Media
- Impacto: Alto (datos inconsistentes)
- Respuesta: Point-in-time recovery en misma región

### Escenario C: Ataque DDoS masivo
- Probabilidad: Media
- Impacto: Alto (degradación severa)
- Respuesta: AWS Shield Advanced + failover si es crítico

## 3. Procedimiento de Failover (Escenario A)
1. [ ] **DETECCIÓN** (automático)
   - CloudWatch Alarm: "PrimaryRegionUnhealthy"
   - PagerDuty notifica al on-call

2. [ ] **VERIFICACIÓN** (arquitecto on-call, 5 min)
   - ¿Es realmente la región o solo nuestro servicio?
   - Verificar AWS Status Dashboard
   - Verificar health checks en us-west-2

3. [ ] **DECISIÓN GO/NO-GO** (arquitecto + VP, 5 min)
   - Si región caída >10 min → GO
   - Si región volviendo → esperar

4. [ ] **EJECUCIÓN** (arquitecto + DevOps, 15 min)
   a. terraform apply -var="dr_active=true"  (us-west-2)
   b. aws rds promote-read-replica --db-instance-id shopflow-dr
   c. Actualizar Route 53 weighted routing a us-west-2 = 100%
   d. Verificar: curl https://api.shopflow.com/health

5. [ ] **VALIDACIÓN** (QA on-call, 10 min)
   - Smoke tests automatizados
   - Checkout de prueba

6. [ ] **COMUNICACIÓN** (VP, 5 min)
   - Status page update
   - Email a clientes enterprise
   - Slack interno #incidents

## 4. Procedimiento de Rollback (cuando primaria vuelve)
1. Sincronizar datos de DR → primaria.
2. Revertir Route 53 a us-east-1.
3. terraform apply -var="dr_active=false" (DR vuelve a standby).

## 5. RPO/RTO Objetivo vs Real
| Métrica | Objetivo | Último Test |
|---------|----------|-------------|
| RPO     | <5 min   | 45 seg      |
| RTO     | <30 min  | 23 min      |

## 6. Calendario de Tests
- Test de failover: Trimestral
- Test de restore de backup: Mensual
- Simulacro completo: Semestral
```

## 35.7 Pruebas de DR: Sin Test No Hay Plan

> "Un plan de disaster recovery que no se ha probado es un cuento de hadas."

### Tipos de Tests

```
1. Tabletop Exercise (Mesa):
   Reúnes al equipo, hablas el escenario paso a paso.
   ¿Quién llama a quién? ¿Qué documentación falta?
   Frecuencia: Mensual. Duración: 1 hora.

2. Simulación Parcial:
   Restauras un componente (solo BD, solo app).
   Sin afectar producción.
   Frecuencia: Mensual.

3. Simulación Completa:
   Failover real a DR. Tráfico real va a DR.
   Viernes noche o fin de semana.
   Frecuencia: Trimestral/semestral. Duración: 4-8 horas.

4. Chaos Engineering (avanzado):
   Apagas producción en horario laboral (!!!).
   Netflix lo hace. Tú probablemente no deberías... aún.
```

## 35.8 Errores Comunes que Vi en Mi Carrera

```
1. "Tenemos backups" pero nunca los restauraron.
   → El backup se corrompió hace 3 meses y nadie lo sabía.

2. "DR está en otra región" configurado manualmente.
   → La persona que lo configuró renunció. Nadie sabe cómo funciona.

3. "DNS failover es automático" pero el TTL es de 24 horas.
   → El failover técnicamente funciona, pero los clientes apuntan
     a la IP vieja por 24 horas más.

4. "La documentación está en la wiki" en la misma región caída.
   → No puedes leer el manual de emergencia si está en el
     sistema caído. Siempre copia offline.

5. "Probamos DR en staging, no en producción."
   → Staging tenía la mitad de datos y configuraciones diferentes.
     Producción falló distinto.

6. "El failover es automático" y un falso positivo lo activó.
   → 2 AM. Un pico de latencia disparó el failover automático.
     Producción migró a DR. Los datos se desincronizaron.
     Pesadilla de 8 horas para volver.

   Lección: Failover automático es peligroso sin excelente
            health checking. Mejor semi-automático (humano decide).
```

## 35.9 Checklist de Preparación DR

- [ ] ¿Tienes backups automatizados y verificados regularmente?
- [ ] ¿Restauraste un backup completo en los últimos 30 días?
- [ ] ¿Tus backups están en otra región física?
- [ ] ¿Tu infraestructura como código puede recrear TODO desde cero?
- [ ] ¿El DR Plan está en un lugar accesible durante un outage? (No en la misma infraestructura)
- [ ] ¿Hiciste un simulacro de DR en los últimos 6 meses?
- [ ] ¿Sabes exactamente quién debe ser contactado y cómo?
- [ ] ¿Tus RPO/RTO fueron definidos con el negocio, no solo con ingeniería?
- [ ] ¿Documentaste las dependencias externas que también deben recuperarse?
- [ ] ¿El costo de DR está en el presupuesto anual, no como sorpresa?

---

> **Reflexión del capítulo**: El disaster recovery es como un seguro de vida: esperas no necesitarlo, pero si lo necesitas y no lo tienes, las consecuencias son catastróficas. Tu trabajo como arquitecto es asegurarte de que cuando (no si) el desastre ocurre, el negocio pueda continuar. No es el tema más glamoroso de la arquitectura, pero es el que más impacto puede tener en tu carrera y en la supervivencia de la empresa.
