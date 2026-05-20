# Capítulo 8: Orquestación con Swarm y Kubernetes

---

> *"Un solo Docker es poderoso. Un ejército de Dockers orquestados es imparable."*

---

## 8.0 Introducción

Has aprendido a construir imágenes, a correr contenedores, a enlazarlos con redes y a persistir datos con volúmenes. Has dominado Docker Compose y eres capaz de levantar stacks completos de microservicios con un solo comando. Todo corre en tu máquina. Todo funciona. Todo está bajo control... hasta que deja de estarlo.

La pregunta inevitable llega cuando tu aplicación sale del laboratorio y se enfrenta al mundo real: **¿qué pasa cuando el host Docker se cae?** La respuesta es brutal: todo se cae contigo. No hay magia. No hay failover. No hay alta disponibilidad. Tu aplicación, tus bases de datos, tu API, tu frontend... todo desaparece en un instante.

Este capítulo aborda exactamente ese problema. Vamos a cruzar el abismo que separa el Docker de desarrollo del Docker de producción. Vamos a hablar de **orquestación**: el arte y la ciencia de gestionar contenedores a través de múltiples máquinas de forma coordinada, resiliente y automatizada.

Cubriremos dos plataformas:
- **Docker Swarm**: el orquestador nativo de Docker, integrado en el motor, sorprendentemente simple de configurar y perfecto para equipos que quieren alta disponibilidad sin complejidad innecesaria.
- **Kubernetes (K8s)**: el estándar de facto de la industria, el proyecto open-source más grande después de Linux, una plataforma masiva y extensible que domina el cloud computing moderno.

Al final de este capítulo, desplegarás la misma aplicación de 3 capas en ambas plataformas, compararás la experiencia, y tendrás criterio propio para decidir cuándo usar cada una.

---

## 8.1 ¿Por qué orquestación? Los límites de Docker standalone

Antes de lanzarnos a la orquestación, entendamos qué problemas resuelve. Porque si no tienes estos problemas, quizás no necesitas orquestar. Pero si los tienes, la orquestación es inevitable.

### 8.1.1 Una máquina = límite de recursos

En Docker standalone (un solo host), todos tus contenedores comparten los recursos de una única máquina física o virtual. CPU, memoria RAM, disco — todo lo que tienes es lo que esa máquina ofrece. Cuando tu aplicación crece y necesita más, tienes dos opciones:

1. **Escalado vertical** (scale up): comprar una máquina más grande. Llega un punto donde no existen máquinas suficientemente grandes para tu carga de trabajo, o el costo se vuelve prohibitivo.
2. **Escalado horizontal** (scale out): añadir más máquinas. Pero Docker standalone no tiene mecanismos nativos para distribuir contenedores entre múltiples hosts.

```
┌─────────────────────────────────┐
│  DOCKER STANDALONE              │
│                                 │
│  ┌───────┐ ┌───────┐ ┌───────┐ │
│  │  Web  │ │  API  │ │  DB   │ │
│  │ Cont. │ │ Cont. │ │ Cont. │ │
│  └───────┘ └───────┘ └───────┘ │
│                                 │
│  CPU: 8 cores    RAM: 32 GB    │
│  Disco: 500 GB                 │
│                                 │
│  TODO comparte la misma máquina│
│  Si se llena el disco, TODO    │
│  se detiene.                   │
└─────────────────────────────────┘
```

Con orquestación, cada nodo aporta sus recursos al cluster. El orquestador decide dónde colocar cada contenedor. Puedes añadir 100 nodos si hace falta.

```
┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│   Nodo 1     │  │   Nodo 2     │  │   Nodo 3     │
│              │  │              │  │              │
│ ┌───────┐    │  │ ┌───────┐    │  │ ┌───────┐    │
│ │  Web  │    │  │ │  Web  │    │  │ │  DB   │    │
│ │ Cont. │    │  │ │ Cont. │    │  │ │ Cont. │    │
│ └───────┘    │  │ └───────┘    │  │ └───────┘    │
│ ┌───────┐    │  │ ┌───────┐    │  │              │
│ │  API  │    │  │ │  API  │    │  │              │
│ │ Cont. │    │  │ │ Cont. │    │  │              │
│ └───────┘    │  │ └───────┘    │  │              │
│              │  │              │  │              │
│ CPU: 4 cores │  │ CPU: 4 cores │  │ CPU: 4 cores │
│ RAM: 8 GB   │  │ RAM: 8 GB   │  │ RAM: 8 GB   │
└──────────────┘  └──────────────┘  └──────────────┘
```

### 8.1.2 Sin alta disponibilidad (HA)

En Docker standalone, si el host físico se apaga, se cuelga, o alguien tropieza con el cable de red, todos tus servicios dejan de funcionar. No hay un "plan B". No hay un nodo de respaldo esperando para tomar el relevo. El downtime es igual al tiempo que tardas en darte cuenta + resolver el problema + reiniciar todo.

Un orquestador monitoriza constantemente el estado del cluster. Si un nodo cae, los contenedores que corrían en él son reprogramados automáticamente en otros nodos con recursos disponibles. La aplicación sigue funcionando — quizás con capacidad reducida, pero sigue funcionando.

```
                   ┌──────────────────────┐
                   │   ORQUESTADOR        │
                   │                      │
                   │ Estado deseado:      │
                   │ web=3, api=3, db=1   │
                   └──────────┬───────────┘
                              │
          ┌───────────────────┼───────────────────┐
          │                   │                   │
   ┌──────▼──────┐    ┌──────▼──────┐    ┌──────▼──────┐
   │   Nodo 1    │    │   Nodo 2    │    │   Nodo 3    │
   │   ACTIVO    │    │   ACTIVO    │    │   CAÍDO!    │
   │             │    │             │    │             │
   │ web:1       │    │ web:1       │    │ web:1 ✗     │
   │ api:1       │    │ api:1       │    │ api:1 ✗     │
   │ db:1        │    │             │    │             │
   └─────────────┘    └─────────────┘    └─────────────┘
                              │
                              │ REPROGRAMA EN NODO 2:
                              │
                     ┌────────▼────────┐
                     │    Nodo 2        │
                     │  web:1 → web:2   │
                     │  api:1 → api:2   │
                     └──────────────────┘
```

### 8.1.3 Sin balanceo de carga automático entre hosts

En Docker standalone, si quieres exponer un servicio al exterior, publicas un puerto (`-p 8080:80`). El tráfico llega a ese host y a ese contenedor. Si tienes 3 instancias del mismo servicio corriendo en 3 hosts diferentes, necesitas un balanceador de carga externo (HAProxy, Nginx, un Load Balancer de cloud) para distribuir el tráfico entre ellas. Y ese balanceador es otro punto único de fallo.

Un orquestador incluye un **routing mesh** integrado: cualquier nodo del cluster puede recibir tráfico para cualquier servicio, y automáticamente lo reenvía al nodo donde realmente está corriendo el contenedor. Esto significa que puedes apuntar tu DNS a cualquier nodo del cluster y el tráfico llegará a destino.

```
CLIENTE ────► cualquiera de estos nodos ────► contenedor correcto

     ┌──────────┐      ┌──────────┐      ┌──────────┐
     │  Nodo 1  │      │  Nodo 2  │      │  Nodo 3  │
     │          │      │          │      │          │
     │ web:1 ◄──┼──────┼──────────┼──────┤          │
     │          │      │          │      │          │
     └──────────┘      └──────────┘      └──────────┘
          ▲                                  ▲
          │                                  │
          └──── CLIENTE ─────────────────────┘
               (puede conectarse a cualquier nodo)
```

### 8.1.4 Sin despliegues graduales (rolling updates)

Actualizar una aplicación en producción con Docker standalone es manual: detienes el contenedor viejo, arrancas el nuevo, esperas que esté listo, luego pasas al siguiente. Si algo sale mal, a rezar.

Un orquestador ofrece **rolling updates** nativos: reemplaza las instancias una por una, de forma controlada, verificando que cada nueva instancia esté saludable antes de continuar. Si algo falla, puedes hacer **rollback** con un solo comando y volver a la versión anterior.

```
Versión v1 (3 réplicas) ──► Versión v2 (3 réplicas)

Antes:         Durante:        Después:
[web:v1]      [web:v2]        [web:v2]
[web:v1]  →   [web:v1]   →    [web:v2]
[web:v1]      [web:v1]        [web:v2]

        Una por una.
        Sin downtime.
        Con posibilidad de rollback.
```

### 8.1.5 Sin secretos nativos

En Docker standalone, las contraseñas y tokens suelen pasarse como variables de entorno (`-e DB_PASSWORD=supersecreto`). Esto tiene problemas graves:

- Aparecen en texto plano en `docker inspect`.
- Se transmiten sin encriptar.
- No hay rotación automática.
- Cualquiera con acceso al host puede leerlas.

Un orquestador ofrece **secrets management** nativo: los secretos se almacenan encriptados en el cluster, se montan como archivos en memoria dentro del contenedor (nunca en disco), y solo los contenedores autorizados pueden acceder a ellos.

```
┌──────────────────────────────────────────┐
│         GESTIÓN DE SECRETOS              │
│                                          │
│  docker secret create db_pass -          │
│  ┌──────────────────────────┐           │
│  │  Raft Log Encriptado     │           │
│  │  ┌─────────────────────┐ │           │
│  │  │ db_pass: AES-256... │ │           │
│  │  └─────────────────────┘ │           │
│  └──────────────────────────┘           │
│                                          │
│  Contenedor accede vía:                  │
│  /run/secrets/db_pass                    │
│                                          │
│  El archivo solo existe en tmpfs (RAM).  │
│  No persiste en disco.                   │
└──────────────────────────────────────────┘
```

---

## 8.2 Docker Swarm — PARTE 1: ARQUITECTURA

Docker Swarm es el orquestador nativo incluido en Docker Engine desde la versión 1.12 (julio 2016). No requiere instalación adicional. No requiere software externo. Está ahí, esperando ser activado.

### 8.2.1 ¿Qué es Swarm?

Swarm (enjambre) convierte un grupo de máquinas con Docker en un único **cluster** que se gestiona como si fuera un solo host Docker. El cluster completo puede aceptar comandos `docker` normales y Swarm se encarga de distribuir el trabajo entre los nodos.

La metáfora del enjambre es acertada: cada abeja (nodo) es autónoma, pero el enjambre completo actúa como un organismo coordinado. Si una abeja muere, el enjambre sigue funcionando.

```
┌─────────────────────────────────────────────────────┐
│                   DOCKER SWARM CLUSTER               │
│                                                     │
│  ┌─────────────────┐    ┌─────────────────┐         │
│  │   Manager 1     │    │   Manager 2     │         │
│  │   (LÍDER)       │◄──►│   (FOLLOWER)    │         │
│  │                 │    │                 │         │
│  │ Toma decisiones │    │ Respaldo del    │         │
│  │ Distribuye tareas│   │ estado Raft     │         │
│  └────────┬────────┘    └─────────────────┘         │
│           │                                         │
│           │ Asigna tareas a workers                 │
│           │                                         │
│  ┌────────▼────────┐    ┌─────────────────┐         │
│  │   Worker 1      │    │   Worker 2      │         │
│  │                 │    │                 │         │
│  │ ┌──────┐        │    │ ┌──────┐        │         │
│  │ │web:1 │        │    │ │web:2 │        │         │
│  │ └──────┘        │    │ └──────┘        │         │
│  │ ┌──────┐        │    │ ┌──────┐        │         │
│  │ │api:1 │        │    │ │api:2 │        │         │
│  │ └──────┘        │    │ └──────┘        │         │
│  └─────────────────┘    └─────────────────┘         │
│                                                     │
└─────────────────────────────────────────────────────┘
```

### 8.2.2 Manager Nodes

Los **manager nodes** son el cerebro del cluster. Son responsables de:

- **Mantener el estado del cluster**: saben cuántos nodos hay, cuáles están activos, qué servicios están definidos, cuántas réplicas debe haber.
- **Orquestar**: reciben las definiciones de servicios y las convierten en tareas concretas que asignan a los workers.
- **Programar (Scheduling)**: deciden en qué nodo ejecutar cada tarea basándose en constraints, recursos disponibles y afinidad.
- **Alta disponibilidad**: mediante el algoritmo de consenso Raft, los managers mantienen una copia consistente del estado del cluster.

**IMPORTANTE**: Un manager también puede ser worker. De hecho, en un cluster pequeño (1-3 nodos), todos los nodos suelen ser managers. Pero en producción es recomendable dedicar los managers exclusivamente a tareas de gestión.

```
┌─────────────────────────────────────────┐
│          MANAGER NODE                   │
│                                         │
│  ┌─────────────────────────────────┐    │
│  │     ORCHESTRATOR (SwarmKit)     │    │
│  │                                 │    │
│  │  ┌─────────┐  ┌──────────┐     │    │
│  │  │Dispatcher│  │Scheduler │     │    │
│  │  └────┬─────┘  └────┬─────┘     │    │
│  │       │              │           │    │
│  │       │    ┌─────────▼──────┐    │    │
│  │       │    │  Allocator     │    │    │
│  │       │    └────────────────┘    │    │
│  │       │                          │    │
│  │  ┌────▼──────────────────────┐   │    │
│  │  │    Raft Consensus Store   │   │    │
│  │  │    (BoltDB embebido)      │   │    │
│  │  └───────────────────────────┘   │    │
│  └─────────────────────────────────┘    │
│                                         │
│  ┌─────────────────────────────────┐    │
│  │     DOCKER ENGINE               │    │
│  │     (puede correr contenedores  │    │
│  │      si no está en drain)       │    │
│  └─────────────────────────────────┘    │
└─────────────────────────────────────────┘
```

### 8.2.3 Worker Nodes

Los **worker nodes** son los que ejecutan el trabajo real. Reciben instrucciones de los managers y reportan su estado de vuelta. Un worker:

- Corre los contenedores que el manager le asigna.
- Reporta el estado de las tareas (running, failed, completed).
- Reporta sus recursos disponibles (CPU, memoria, disco).
- No participa en las decisiones del cluster.
- No puede crear o modificar servicios (solo ejecutar).

Si un worker cae, el manager lo detecta (por falta de heartbeats) y reprograma sus tareas en otros workers disponibles.

```
┌─────────────────────────────────────────┐
│          WORKER NODE                    │
│                                         │
│  ┌─────────────────────────────────┐    │
│  │     DOCKER ENGINE               │    │
│  │                                 │    │
│  │  ┌──────────────────────────┐   │    │
│  │  │     Executor             │   │    │
│  │  │  ┌────────┐ ┌────────┐   │   │    │
│  │  │  │Task:   │ │Task:   │   │   │    │
│  │  │  │web.1   │ │api.3   │   │   │    │
│  │  │  └────────┘ └────────┘   │   │    │
│  │  └──────────────────────────┘   │    │
│  └─────────────────────────────────┘    │
│                                         │
│  ┌─────────────────────────────────┐    │
│  │     AGENTE (SwarmKit)           │    │
│  │     - Recibe asignaciones       │    │
│  │     - Reporta estado            │    │
│  │     - Envía heartbeats          │    │
│  └─────────────────────────────────┘    │
│                                         │
└─────────────────────────────────────────┘
```

### 8.2.4 Número recomendado de managers

El algoritmo de consenso Raft requiere **quorum** (mayoría) para tomar decisiones. Esto implica:

- Con **1 manager**: sin HA. Si ese manager cae, el cluster no puede tomar decisiones (aunque los contenedores que ya están corriendo siguen funcionando).
- Con **2 managers**: PEOR que 1. Si uno cae, no hay quorum (1 de 2 no es mayoría). El cluster se congela.
- Con **3 managers**: tolera la caída de 1. Ideal para producción pequeña.
- Con **5 managers**: tolera la caída de 2. Producción mediana.
- Con **7 managers**: tolera la caída de 3. Límite práctico recomendado.

> **Regla de oro**: Siempre número IMPAR. N = 2F + 1, donde F es el número de fallos que toleras.

```
Tabla de tolerancia a fallos en managers:

+------------+------------------+-------------------+
| Managers   | Quorum necesario | Fallos tolerados  |
+------------+------------------+-------------------+
|     1      |        1         |        0          |
|     2      |        2         |        0 (¡PEOR!) |
|     3      |        2         |        1          |
|     5      |        3         |        2          |
|     7      |        4         |        3          |
|     9      |        5         |        4 (evitar) |
+------------+------------------+-------------------+
```

### 8.2.5 Raft Consensus: cómo Swarm mantiene el estado

**Raft** es un algoritmo de consenso diseñado para ser entendible (a diferencia de Paxos). En Swarm, el Raft log almacena todo el estado del cluster: qué nodos hay, qué servicios, cuántas réplicas, qué secretos, todo.

```
Funcionamiento de Raft en Swarm:

PASO 1: El líder recibe una petición
  │
  │  docker service create --replicas 3 --name web nginx
  │
  ▼
┌─────────────────────────────────────────────────────┐
│               MANAGER 1 (LÍDER)                     │
│                                                     │
│  1. Recibe la orden de crear servicio               │
│  2. Escribe la entrada en su Raft log               │
│  3. Envía la entrada a los followers                │
└──────────────────┬──────────────────────────────────┘
                   │
    ┌──────────────┼──────────────┐
    │              │              │
    ▼              ▼              ▼
┌─────────┐  ┌─────────┐  ┌─────────┐
│Manager 2│  │Manager 3│  │Manager 4│
│FOLLOWER │  │FOLLOWER │  │FOLLOWER │
│         │  │         │  │         │
│OK ✓     │  │OK ✓     │  │OK ✓     │
└─────────┘  └─────────┘  └─────────┘
    │              │              │
    └──────────────┼──────────────┘
                   │
    Quorum alcanzado: 3 de 4 = mayoría ✓
                   │
                   ▼
    ┌──────────────────────────────┐
    │ La entrada se COMMITEA       │
    │ El líder distribuye las      │
    │ tareas a los workers         │
    └──────────────────────────────┘
```

**Elección de líder**: Si el líder actual falla (no envía heartbeats), los followers inician una elección. El que recibe mayoría de votos se convierte en el nuevo líder. Todo esto ocurre en milisegundos y es transparente para los servicios.

```
CRONOLOGÍA DE ELECCIÓN DE LÍDER:

t=0ms:    Líder (Manager 1) envía heartbeat cada 200ms.
t=200ms:  Heartbeat recibido. Todo normal.
t=400ms:  Heartbeat recibido. Todo normal.
t=500ms:  Manager 1 CAE.
t=600ms:  No hay heartbeat. Managers 2,3,4 esperan.
t=800ms:  Timeout de elección aleatorio.
          Manager 2 pide votos primero.
t=900ms:  Manager 3 y 4 votan por Manager 2.
          Manager 2 es el nuevo LÍDER.
t=1000ms: Nuevo líder empieza a enviar heartbeats.
          Cluster operativo. Downtime de gestión: ~500ms.
          Servicios siguen funcionando sin interrupción.
```

### 8.2.6 Inicializar Swarm

Llega el momento de la verdad. Vamos a crear nuestro primer cluster Swarm.

**Prerrequisitos**:
- 3 máquinas (físicas o virtuales) con Docker Engine instalado (18.06+).
- Puertos abiertos entre ellas:
  - **2377/TCP**: comunicación de gestión del cluster (managers).
  - **7946/TCP y 7946/UDP**: comunicación entre nodos (gossip network).
  - **4789/UDP**: tráfico de red overlay (VXLAN).

```
┌──────────────────────────────────────────────────┐
│              PUERTOS DE SWARM                    │
│                                                  │
│  2377/TCP  →  API de gestión del cluster        │
│               (solo managers)                    │
│                                                  │
│  7946/TCP  →  Gossip protocol (descubrimiento   │
│  7946/UDP     de nodos, heartbeats)              │
│                                                  │
│  4789/UDP  →  Tráfico overlay network (VXLAN)   │
│               (datos entre contenedores          │
│                en distintos hosts)               │
└──────────────────────────────────────────────────┘
```

Vamos a usar 3 VMs para este ejemplo:

| Rol     | Hostname    | IP           |
|---------|-------------|--------------|
| Manager | manager-01  | 192.168.1.10 |
| Worker  | worker-01   | 192.168.1.20 |
| Worker  | worker-02   | 192.168.1.30 |

**Paso 1: Inicializar el Swarm en el manager**

```bash
# En manager-01 (192.168.1.10)
docker swarm init --advertise-addr 192.168.1.10
```

Salida esperada:

```
Swarm initialized: current node (abc123def456) is now a manager.

To add a worker to this swarm, run the following command:

    docker swarm join --token SWMTKN-1-xxxxxxxxxxxxxxxx-xxxxxxxxxxxxxxxx \
    192.168.1.10:2377

To add a manager to this swarm, run 'docker swarm join-token manager'
and follow the instructions.
```

**Análisis del comando**:

- `docker swarm init`: inicializa un nuevo Swarm. Este nodo se convierte en el primer manager (y líder automáticamente).
- `--advertise-addr 192.168.1.10`: la IP que este nodo publica al cluster. Debe ser accesible desde los otros nodos. Si tienes múltiples interfaces de red, especifica cuál usar.

**Paso 2: Verificar que el nodo es el manager**

```bash
docker node ls
```

Salida:

```
ID                            HOSTNAME    STATUS    AVAILABILITY   MANAGER STATUS
abc123def456 *                manager-01  Ready     Active         Leader
```

El asterisco `*` indica que estamos conectados a ese nodo. `Leader` significa que es el líder actual de Raft.

**Paso 3: Obtener tokens de join**

Los tokens son las "llaves" que permiten a nuevos nodos unirse al cluster. Hay dos tipos:

```bash
# Token para workers
docker swarm join-token worker

# Token para managers
docker swarm join-token manager
```

Cada comando imprime el comando completo que debe ejecutarse en el nuevo nodo.

**Paso 4: Unir workers al cluster**

```bash
# En worker-01 (192.168.1.20)
docker swarm join --token SWMTKN-1-xxxxxxxxxxxxxxxx-xxxxxxxxxxxxxxxx \
  192.168.1.10:2377
```

Salida:

```
This node joined a swarm as a worker.
```

```bash
# En worker-02 (192.168.1.30)
docker swarm join --token SWMTKN-1-xxxxxxxxxxxxxxxx-xxxxxxxxxxxxxxxx \
  192.168.1.10:2377
```

**Paso 5: Verificar el cluster**

```bash
# En manager-01
docker node ls
```

Salida:

```
ID                            HOSTNAME    STATUS    AVAILABILITY   MANAGER STATUS
abc123def456 *                manager-01  Ready     Active         Leader
ghi789jkl012                  worker-01   Ready     Active
mno345pqr678                  worker-02   Ready     Active
```

¡Cluster Swarm de 3 nodos operativo!

### 8.2.7 Inspeccionar y gestionar nodos

**`docker node inspect`**: información detallada de un nodo.

```bash
docker node inspect worker-01
```

Este comando devuelve un JSON enorme con:

```json
[
  {
    "ID": "ghi789jkl012",
    "Version": { "Index": 9 },
    "CreatedAt": "2026-05-20T10:00:00.000000000Z",
    "UpdatedAt": "2026-05-20T10:05:00.000000000Z",
    "Spec": {
      "Labels": {},
      "Role": "worker",
      "Availability": "active"
    },
    "Description": {
      "Hostname": "worker-01",
      "Platform": {
        "Architecture": "x86_64",
        "OS": "linux"
      },
      "Resources": {
        "NanoCPUs": 4000000000,
        "MemoryBytes": 8270000000
      },
      "Engine": {
        "EngineVersion": "24.0.0",
        "Plugins": [
          { "Type": "Network", "Name": "overlay" },
          { "Type": "Volume", "Name": "local" }
        ]
      }
    },
    "Status": {
      "State": "ready",
      "Addr": "192.168.1.20"
    },
    "ManagerStatus": null
  }
]
```

Podemos extraer campos específicos con `--format` de Go templates:

```bash
# Solo el rol
docker node inspect --format '{{ .Spec.Role }}' worker-01
# Salida: worker

# Solo los recursos de CPU (en nanosegundos de CPU)
docker node inspect --format '{{ .Description.Resources.NanoCPUs }}' worker-01
# Salida: 4000000000  (4 CPUs)

# Solo el estado
docker node inspect --format '{{ .Status.State }}' worker-01
# Salida: ready
```

**`docker node update`**: modificar las propiedades de un nodo.

```bash
# Añadir una etiqueta (label) al nodo para constraints
docker node update --label-add region=us-east --label-add env=production worker-01

# Añadir múltiples labels
docker node update --label-add tier=frontend worker-02

# Verificar labels
docker node inspect --format '{{ .Spec.Labels }}' worker-01
# Salida: map[env:production region:us-east]
```

Las labels son fundamentales para las **placement constraints** que veremos más adelante.

### 8.2.8 Promover y degradar nodos

```bash
# Promover un worker a manager
docker node promote worker-01

# Degradar un manager a worker
docker node demote worker-01
```

**Promover** a manager significa que ese nodo ahora:
- Participa en el consenso Raft.
- Recibe una copia completa del Raft log.
- Puede convertirse en líder.
- Consume más recursos (CPU, memoria, disco para el log).

**Degradar** lo contrario: deja de participar en Raft, se convierte en worker puro.

```bash
# Ejemplo: promover worker-01 para tener 2 managers
docker node promote worker-01

docker node ls
```

Salida:

```
ID                            HOSTNAME    STATUS    AVAILABILITY   MANAGER STATUS
abc123def456 *                manager-01  Ready     Active         Leader
ghi789jkl012                  worker-01   Ready     Active         Reachable
mno345pqr678                  worker-02   Ready     Active
```

`Reachable` significa que worker-01 es manager (follower) y está sincronizado con el líder.

### 8.2.9 Drain: poner un nodo en mantenimiento

A veces necesitas hacer mantenimiento a un nodo (actualizar el SO, cambiar hardware, etc.). Con `drain`, le dices a Swarm:

> "No programes más tareas en este nodo, y mueve las existentes a otros nodos."

```bash
# Poner worker-02 en drain
docker node update --availability drain worker-02

docker node ls
```

Salida:

```
ID                            HOSTNAME    STATUS    AVAILABILITY   MANAGER STATUS
abc123def456 *                manager-01  Ready     Active         Leader
ghi789jkl012                  worker-01   Ready     Active         Reachable
mno345pqr678                  worker-02   Ready     Drain
```

Cualquier contenedor corriendo en worker-02 será **detenido y reprogramado** en otro nodo con `Active`. Las réplicas de servicios se mantienen.

```bash
# Volver a activar el nodo después del mantenimiento
docker node update --availability active worker-02
```

### 8.2.10 Salir de Swarm

```bash
# Un worker abandona el cluster
docker swarm leave

# Un manager abandona el cluster (requiere --force)
docker swarm leave --force
```

Si un manager sale con `--force`, se lleva su copia del Raft log. Si era el líder, se desencadena una nueva elección. Si el quorum se pierde (demasiados managers salen), el cluster deja de aceptar cambios hasta que se restaure el quorum o se haga recuperación forzada.

### 8.2.11 Recuperación de desastres (Disaster Recovery)

Si todos los managers de un Swarm caen y no hay quorum, el cluster entra en modo "solo lectura": los servicios siguen corriendo, pero no puedes crear, modificar o eliminar nada.

**Recuperación forzada**:

```bash
# PASO 1: Parar Docker en el manager que quieres recuperar
systemctl stop docker

# PASO 2: Forzar la creación de un nuevo cluster
#         desde el estado existente
docker swarm init --force-new-cluster \
  --advertise-addr 192.168.1.10

# PASO 3: Los workers se reconectan automáticamente
```

**Respaldo preventivo**: Swarm guarda su estado en `/var/lib/docker/swarm/`. Puedes respaldar este directorio periódicamente:

```bash
# En un manager
systemctl stop docker
tar -czf swarm-backup-$(date +%Y%m%d).tar.gz \
  /var/lib/docker/swarm/
systemctl start docker
```

La restauración se hace copiando el backup en un nuevo nodo y ejecutando `docker swarm init --force-new-cluster`.

---

## 8.3 Docker Swarm — PARTE 2: SERVICES

Hasta ahora tenemos un cluster: 3 nodos que se conocen entre sí. Pero no hay nada corriendo. Es hora de desplegar servicios.

### 8.3.1 ¿Qué es un Service?

En Swarm, un **service** (servicio) es la definición abstracta de una tarea que quieres ejecutar. No es un contenedor concreto. Es la **especificación** de:

- Qué imagen usar.
- Cuántas réplicas deseas.
- En qué puertos exponer.
- En qué red overlay comunicarse.
- Qué volúmenes montar.
- Qué constraints de ubicación aplicar.
- Qué límites de recursos imponer.
- Política de reinicio.
- Estrategia de actualización.

Cuando creas un service, Swarm lo traduce en **tasks** (tareas) concretas. Cada task es una unidad de ejecución que el scheduler asigna a un nodo. Cada task ejecuta exactamente un contenedor.

```
┌──────────────────────────────────────────────────┐
│  SERVICE: web                                    │
│  Imagen: nginx:1.25                              │
│  Réplicas: 3                                     │
│  Puerto: 80                                      │
└─────────────┬────────────────────────────────────┘
              │
              │ Swarm descompone en Tasks
              │
    ┌─────────┼─────────┐
    │         │         │
    ▼         ▼         ▼
┌────────┐┌────────┐┌────────┐
│ web.1  ││ web.2  ││ web.3  │
│ Task   ││ Task   ││ Task   │──── Tareas
└───┬────┘└───┬────┘└───┬────┘
    │         │         │
    ▼         ▼         ▼
┌────────┐┌────────┐┌────────┐
│nginx:1 ││nginx:1 ││nginx:1 │──── Contenedores
│worker-1││worker-2││worker-1│
└────────┘└────────┘└────────┘
```

### 8.3.2 `docker service create`

El comando más completo de Swarm. Su sintaxis es similar a `docker run`, pero con esteroides.

```bash
docker service create \
  --name web \
  --replicas 3 \
  --publish published=8080,target=80 \
  --network frontend \
  --mount type=volume,source=web-data,target=/usr/share/nginx/html \
  --constraint node.role==worker \
  --limit-cpu 0.5 \
  --limit-memory 512m \
  --reserve-cpu 0.25 \
  --reserve-memory 256m \
  --restart-condition on-failure \
  --restart-delay 5s \
  --restart-max-attempts 3 \
  --update-delay 10s \
  --update-parallelism 1 \
  --update-failure-action rollback \
  --env NGINX_HOST=example.com \
  --env NGINX_PORT=80 \
  nginx:1.25
```

Analicemos cada opción:

| Opción | Significado |
|--------|-------------|
| `--name web` | Nombre del servicio. Debe ser único en el cluster. |
| `--replicas 3` | Número de instancias (tareas) que mantener corriendo. |
| `--publish published=8080,target=80` | Expone el puerto 8080 en el routing mesh, que mapea al puerto 80 del contenedor. |
| `--network frontend` | Conecta el servicio a la red overlay `frontend`. |
| `--mount type=volume,source=web-data,target=/usr/share/nginx/html` | Monta el volumen `web-data` en el path indicado. |
| `--constraint node.role==worker` | Solo colocar tareas en nodos worker. |
| `--limit-cpu 0.5` | Límite máximo de CPU: 0.5 cores (50% de un core). |
| `--limit-memory 512m` | Límite máximo de RAM: 512 MB. |
| `--reserve-cpu 0.25` | Reserva de CPU (garantizada): 0.25 cores. |
| `--reserve-memory 256m` | Reserva de RAM (garantizada): 256 MB. |
| `--restart-condition on-failure` | Reiniciar el contenedor solo si falla (exit code != 0). |
| `--restart-delay 5s` | Esperar 5 segundos antes de reintentar. |
| `--restart-max-attempts 3` | Máximo 3 reintentos. |
| `--update-delay 10s` | En rolling update, esperar 10s entre cada actualización. |
| `--update-parallelism 1` | Actualizar 1 réplica a la vez. |
| `--update-failure-action rollback` | Si una actualización falla, hacer rollback automático. |
| `--env` | Variables de entorno para el contenedor. |

### 8.3.3 `docker service ls` y `docker service ps`

Ver todos los servicios definidos en el cluster:

```bash
docker service ls
```

Salida:

```
ID             NAME   MODE         REPLICAS   IMAGE        PORTS
abc123def456   web    replicated   3/3        nginx:1.25   *:8080->80/tcp
ghi789jkl012   api    replicated   2/2        api:latest   *:3000->3000/tcp
```

Ver las tareas (réplicas) de un servicio específico:

```bash
docker service ps web
```

Salida:

```
ID             NAME     IMAGE        NODE        DESIRED STATE   CURRENT STATE
def123abc456   web.1    nginx:1.25   worker-01   Running         Running 2 minutes ago
ghi456def789   web.2    nginx:1.25   worker-02   Running         Running 2 minutes ago
jkl789ghi012   web.3    nginx:1.25   manager-01  Running         Running 2 minutes ago
```

Cada línea es una **task**. Observa:
- `DESIRED STATE`: lo que Swarm QUIERE que esté pasando (Running, Shutdown).
- `CURRENT STATE`: lo que REALMENTE está pasando.
- Si `DESIRED STATE` es `Running` pero `CURRENT STATE` es `Failed`, Swarm intentará corregirlo creando una nueva task.

```bash
# Ver solo réplicas que NO están corriendo
docker service ps --filter desired-state=Running web
```

### 8.3.4 `docker service inspect`

```bash
docker service inspect --pretty web
```

Modo `--pretty` da una salida legible:

```
ID:             abc123def456
Name:           web
Service Mode:   Replicated
 Replicas:      3
Placement:
 Constraints:   [node.role == worker]
UpdateConfig:
 Parallelism:   1
 Delay:         10s
 FailureAction: rollback
ContainerSpec:
 Image:         nginx:1.25
 Env:           NGINX_HOST=example.com NGINX_PORT=80
Resources:
 Limits:
  CPU:          0.5
  Memory:       512MiB
 Reservations:
  CPU:          0.25
  Memory:       256MiB
Ports:
 Protocol = tcp
 PublishedPort = 8080
 TargetPort = 80
 PublishMode = ingress
```

Sin `--pretty`, obtienes el JSON completo (útil para scripting).

### 8.3.5 Escalar servicios: `docker service scale`

```bash
# Escalar manualmente a 5 réplicas
docker service scale web=5

# Escalar múltiples servicios a la vez
docker service scale web=5 api=3 worker=10

# Escalar a 0 (quita todas las réplicas, ideal para mantenimiento)
docker service scale web=0
```

Cuando escalas, Swarm inmediatamente crea o destruye tasks para alcanzar el número deseado. Si escalas hacia arriba, el scheduler busca los mejores nodos para las nuevas tasks.

```bash
docker service scale web=10
docker service ps web | grep "Running" | wc -l
# Salida: 10
```

### 8.3.6 `docker service update`

Actualizar cualquier propiedad de un servicio en caliente, sin detenerlo.

```bash
# Cambiar la imagen (esto dispara un rolling update)
docker service update --image nginx:1.26 web

# Cambiar número de réplicas
docker service update --replicas 5 web

# Añadir una variable de entorno
docker service update --env-add DEBUG=true web

# Quitar una variable
docker service update --env-rm DEBUG web

# Cambiar límites de recursos
docker service update --limit-memory 1g --limit-cpu 1 web

# Cambiar el puerto publicado
docker service update --publish-add published=8443,target=443 web

# Cambiar constraints
docker service update --constraint-add 'node.labels.region==eu-west' web

# Cambiar restart policy
docker service update --restart-condition any web

# Forzar un rebalanceo de réplicas (útil si añadiste nodos)
docker service update --force web
```

Cada `docker service update` que cambia la imagen o recursos dispara un **rolling update** con las reglas configuradas.

### 8.3.7 Rolling Updates

Imagina que tienes `web` corriendo con 5 réplicas de `nginx:1.25` y quieres actualizar a `nginx:1.26`.

Con `docker service update --image nginx:1.26 web`, Swarm:

1. **Toma la primera réplica** (web.1) y la detiene.
2. **Crea una nueva réplica** con la imagen `nginx:1.26` en su lugar.
3. **Espera** `--update-delay` (por defecto 0s, recomendado 10s).
4. **Verifica** que la nueva réplica está Running (health check si existe).
5. **Repite** con la siguiente réplica (web.2).
6. **Continúa** hasta actualizar todas.
7. Si alguna **falla**, ejecuta `--update-failure-action` (pause, continue, rollback).

```
ACTUALIZACIÓN GRADUAL (Rolling Update) en acción:

Tiempo →   t=0s          t=10s         t=20s         t=30s         t=40s
         ┌────────┐    ┌────────┐    ┌────────┐    ┌────────┐    ┌────────┐
web.1    │ v1.25  │ →  │ v1.26 ✓│    │ v1.26  │    │ v1.26  │    │ v1.26  │
         └────────┘    └────────┘    └────────┘    └────────┘    └────────┘
         ┌────────┐    ┌────────┐    ┌────────┐    ┌────────┐    ┌────────┐
web.2    │ v1.25  │    │ v1.25  │ →  │ v1.26 ✓│    │ v1.26  │    │ v1.26  │
         └────────┘    └────────┘    └────────┘    └────────┘    └────────┘
         ┌────────┐    ┌────────┐    ┌────────┐    ┌────────┐    ┌────────┐
web.3    │ v1.25  │    │ v1.25  │    │ v1.25  │ →  │ v1.26 ✓│    │ v1.26  │
         └────────┘    └────────┘    └────────┘    └────────┘    └────────┘

         ✓ = réplica nueva verificada como Running
```

**Opciones de rolling update**:

```bash
docker service create \
  --name web \
  --replicas 5 \
  --update-delay 15s \         # Espera 15s entre actualizaciones
  --update-parallelism 2 \     # Actualiza 2 réplicas a la vez
  --update-failure-action rollback \  # Si falla, revierte TODO
  --update-max-failure-ratio 0.3 \    # Tolera hasta 30% de fallos
  --update-order start-first \ # Arranca la nueva ANTES de parar la vieja
  nginx:1.25
```

`--update-order` tiene dos valores:
- `stop-first` (default): para la réplica vieja, luego arranca la nueva. Menor uso de recursos. Posible downtime breve.
- `start-first`: arranca la nueva réplica primero, verifica que esté bien, luego para la vieja. Sin downtime. Mayor uso de recursos temporal.

### 8.3.8 Rollback

Si un rolling update sale mal, puedes volver atrás:

```bash
# Volver a la versión anterior
docker service rollback web
```

El rollback **usa las mismas opciones de update** que configuraste (parallelism, delay, etc.), pero en reversa. Swarm guarda la configuración anterior del servicio.

```bash
# Ver la imagen actual del servicio
docker service inspect --format \
  '{{.Spec.TaskTemplate.ContainerSpec.Image}}' web
```

### 8.3.9 Placement Constraints

Las **constraints** son reglas que le dicen al scheduler DÓNDE puede o no puede colocar las tareas de un servicio.

```bash
# Solo en workers (NO en managers)
docker service create --constraint node.role==worker ...

# Solo en nodos con label específico
docker service create --constraint node.labels.region==us-east ...

# Múltiples constraints (AND lógico)
docker service create \
  --constraint node.role==worker \
  --constraint node.labels.env==production \
  --constraint node.labels.tier==backend \
  ...

# En cualquier nodo EXCEPTO los que tienen cierta label
docker service create --constraint node.labels.env!=staging ...
```

**Placement Preferences** (más flexible que constraints):

```bash
# Distribuir réplicas uniformemente por región
docker service create \
  --placement-pref 'spread=node.labels.region' \
  --replicas 6 \
  nginx
```

Si tienes 3 regiones (`us-east`, `us-west`, `eu-west`) con 2 nodos cada una, Swarm intentará colocar 2 réplicas en cada región.

### 8.3.10 Resource Limits

Especificar límites de recursos es crucial para que el scheduler funcione correctamente:

```bash
docker service create \
  --name api \
  --replicas 3 \
  --limit-cpu 1.5 \          # Máximo 1.5 cores (150% de un core)
  --limit-memory 1024m \     # Máximo 1024 MB de RAM
  --reserve-cpu 0.5 \        # Reserva GARANTIZADA de 0.5 cores
  --reserve-memory 256m \    # Reserva GARANTIZADA de 256 MB
  api:latest
```

- **Reserve**: lo que el contenedor NECESITA. El scheduler solo coloca la tarea en un nodo que tenga al menos esta cantidad disponible.
- **Limit**: lo máximo que el contenedor PUEDE usar. Si excede, Docker lo estrangula (CPU) o lo mata (OOM Killer para memoria).

**Tabla de unidades de recursos**:

| Recurso | Unidades | Ejemplos |
|---------|----------|----------|
| CPU | Núcleos (1 = un core) | `0.5`, `1`, `2`, `1.5` |
| Memoria | Bytes | `256m` (MB), `2g` (GB), `1024k` (KB) |
| NanoCPUs | 1e-9 CPU (interno) | `500000000` = 0.5 cores |

### 8.3.11 Restart Policies

Qué hace Swarm cuando un contenedor muere:

```bash
# Solo reiniciar si falla (código de salida != 0)
docker service create --restart-condition on-failure ...

# Reiniciar siempre (incluso si exit code = 0)
docker service create --restart-condition any ...

# Nunca reiniciar (útil para tareas batch)
docker service create --restart-condition none ...

# Con delay y máximo de intentos
docker service create \
  --restart-condition on-failure \
  --restart-delay 10s \
  --restart-max-attempts 5 \
  --restart-window 60s \  # Ventana de tiempo para contar intentos
  ...
```

### 8.3.12 Modos de Servicio: Replicated vs Global

**Replicated mode** (por defecto): N réplicas distribuidas entre los nodos.

```bash
docker service create --name web --replicas 5 --mode replicated nginx
```

**Global mode**: Exactamente 1 réplica por nodo. Útil para agentes de monitoreo, log shippers, o servicios que deben correr en cada host.

```bash
docker service create \
  --name monitoring-agent \
  --mode global \
  --mount type=bind,source=/var/run/docker.sock,target=/var/run/docker.sock \
  monitoring-agent:latest
```

```
Modo Global en acción:

┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│   Nodo 1     │  │   Nodo 2     │  │   Nodo 3     │
│              │  │              │  │              │
│ ┌──────────┐ │  │ ┌──────────┐ │  │ ┌──────────┐ │
│ │ agent.1  │ │  │ │ agent.1  │ │  │ │ agent.1  │ │
│ └──────────┘ │  │ └──────────┘ │  │ └──────────┘ │
│ ┌──────────┐ │  │ ┌──────────┐ │  │              │
│ │ web.1    │ │  │ │ web.2    │ │  │              │
│ └──────────┘ │  │ └──────────┘ │  │              │
└──────────────┘  └──────────────┘  └──────────────┘

agent → GLOBAL (1 por nodo)
web   → REPLICATED (2 réplicas distribuidas)
```

---

## 8.4 Docker Swarm — PARTE 3: STACKS, SECRETS, CONFIGS

### 8.4.1 Stacks: Docker Compose en Swarm

Un **stack** es la evolución natural de Docker Compose para Swarm. Es exactamente el mismo archivo `docker-compose.yml` que ya conoces, pero con la sección `deploy:` que solo aplica en Swarm.

**Desplegar un stack**:

```bash
docker stack deploy -c docker-compose.yml mystack
```

Donde `mystack` es el nombre que le das al stack. Swarm lo usa como prefijo para todos los recursos (redes, volúmenes, secretos, servicios).

**Ver stacks**:

```bash
docker stack ls
```

**Ver servicios de un stack**:

```bash
docker stack services mystack
```

**Ver tareas de un stack**:

```bash
docker stack ps mystack
```

**Eliminar un stack** (borra TODOS los servicios, redes y secretos del stack):

```bash
docker stack rm mystack
```

### 8.4.2 La sección `deploy:` en Compose para Swarm

Aquí está el archivo de ejemplo completo con todas las opciones de la sección `deploy:`:

```yaml
# docker-compose.yml
version: '3.9'

services:

  # ──── FRONTEND WEB ────
  web:
    image: nginx:1.25
    ports:
      - "80:80"
      - "443:443"
    environment:
      - NGINX_HOST=example.com
      - NGINX_PORT=80
    networks:
      - frontend
    volumes:
      - web_data:/usr/share/nginx/html
    configs:
      - source: nginx_config
        target: /etc/nginx/nginx.conf
    secrets:
      - source: tls_cert
        target: /etc/nginx/certs/server.crt
      - source: tls_key
        target: /etc/nginx/certs/server.key
    deploy:
      mode: replicated
      replicas: 3
      placement:
        constraints:
          - node.role == worker
          - node.labels.region == us-east
        preferences:
          - spread: node.labels.az
      resources:
        limits:
          cpus: '1.5'
          memory: 1024M
        reservations:
          cpus: '0.5'
          memory: 512M
      restart_policy:
        condition: on-failure
        delay: 10s
        max_attempts: 3
        window: 120s
      update_config:
        parallelism: 1
        delay: 15s
        failure_action: rollback
        monitor: 30s
        max_failure_ratio: 0.3
        order: start-first
      rollback_config:
        parallelism: 1
        delay: 5s
        failure_action: pause
        monitor: 30s
        max_failure_ratio: 0.3
      labels:
        - "com.example.description=Frontend web service"
        - "com.example.team=frontend"

  # ──── API REST ────
  api:
    image: api:latest
    ports:
      - "3000:3000"
    environment:
      - NODE_ENV=production
      - DB_HOST=db
    secrets:
      - source: db_password
        target: /run/secrets/db_password
    networks:
      - frontend
      - backend
    depends_on:
      - db
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3000/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s
    deploy:
      mode: replicated
      replicas: 5
      placement:
        constraints:
          - node.role == worker
      resources:
        limits:
          cpus: '1'
          memory: 512M
        reservations:
          cpus: '0.5'
          memory: 256M
      restart_policy:
        condition: any
        delay: 5s
        max_attempts: 5
      update_config:
        parallelism: 2
        delay: 10s
        failure_action: rollback
        order: start-first

  # ──── BASE DE DATOS ────
  db:
    image: postgres:16
    environment:
      - POSTGRES_USER=appuser
      - POSTGRES_DB=appdb
    secrets:
      - source: db_password
        target: /run/secrets/postgres_password
    volumes:
      - db_data:/var/lib/postgresql/data
    networks:
      - backend
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U appuser -d appdb"]
      interval: 10s
      timeout: 5s
      retries: 5
    deploy:
      mode: replicated
      replicas: 1          # Base de datos: 1 réplica (single writer)
      placement:
        constraints:
          - node.labels.tier == database
      resources:
        limits:
          cpus: '2'
          memory: 2048M
        reservations:
          cpus: '1'
          memory: 1024M
      restart_policy:
        condition: any
        delay: 15s
      update_config:
        parallelism: 1
        delay: 30s
        order: stop-first

  # ──── AGENTE DE MONITOREO ────
  monitor:
    image: monitoring-agent:latest
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro
    deploy:
      mode: global           # Exactamente 1 por nodo
      resources:
        limits:
          cpus: '0.25'
          memory: 256M
      restart_policy:
        condition: any

# ──── NETWORKS ────
networks:
  frontend:
    driver: overlay
    attachable: true
    labels:
      - "com.example.tier=frontend"
  backend:
    driver: overlay
    internal: true           # Sin acceso externo (solo entre servicios)
    labels:
      - "com.example.tier=backend"

# ──── VOLUMES ────
volumes:
  web_data:
    driver: local
    labels:
      - "com.example.purpose=web-static"
  db_data:
    driver: local
    labels:
      - "com.example.purpose=database-storage"

# ──── SECRETS ────
secrets:
  db_password:
    external: true           # Secreto creado previamente
  tls_cert:
    external: true
  tls_key:
    external: true

# ──── CONFIGS ────
configs:
  nginx_config:
    file: ./nginx.conf       # Archivo local que se convierte en config
```

### 8.4.3 Secrets: secretos encriptados

Los **secrets** en Swarm son datos sensibles que se almacenan encriptados en el Raft log y se entregan a los contenedores como archivos en un sistema de archivos temporal en RAM (tmpfs).

**Crear un secreto**:

```bash
# Desde stdin (recomendado: no queda en el historial de shell)
echo "MySuperSecretP@ssword123" | docker secret create db_password -

# Desde un archivo
docker secret create api_key ./api_key.txt

# Con una etiqueta para identificarlo
docker secret create \
  --label env=production \
  --label app=api \
  jwt_secret ./jwt_secret.txt
```

**Listar secretos**:

```bash
docker secret ls
```

Salida:

```
ID                          NAME          CREATED             UPDATED
abc123def456ghi789          db_password   2 minutes ago       2 minutes ago
jkl012mno345pqr678          api_key       1 minute ago        1 minute ago
stu901vwx234yz567           jwt_secret    30 seconds ago      30 seconds ago
```

**Inspeccionar un secreto** (no muestra el valor, solo metadatos):

```bash
docker secret inspect db_password
```

```json
[
  {
    "ID": "abc123def456ghi789",
    "Version": { "Index": 42 },
    "CreatedAt": "2026-05-20T12:00:00.000000000Z",
    "UpdatedAt": "2026-05-20T12:00:00.000000000Z",
    "Spec": {
      "Name": "db_password",
      "Labels": {}
    }
  }
]
```

**EL VALOR DEL SECRETO NUNCA SE MUESTRA EN `docker secret inspect`.**

**Usar un secreto en un servicio**:

```bash
docker service create \
  --name api \
  --secret db_password \
  api:latest
```

Por defecto, el secreto se monta en `/run/secrets/db_password`. Puedes personalizar el path:

```bash
docker service create \
  --name api \
  --secret source=db_password,target=/etc/app/secrets/password,mode=0400 \
  api:latest
```

**Dentro del contenedor**:

```bash
# La aplicación lee el secreto como un archivo
cat /run/secrets/db_password
# Salida: MySuperSecretP@ssword123
```

**NO confundas environment con secrets**:

```yaml
# ❌ MAL: Contraseña en variable de entorno
environment:
  - DB_PASSWORD=supersecreto     # Visible en docker inspect

# ✅ BIEN: Contraseña como secreto
secrets:
  - db_password                  # Encriptado, solo accesible vía archivo
environment:
  - DB_PASSWORD_FILE=/run/secrets/db_password  # La app lee la ruta
```

**Rotar un secreto**:

```bash
# 1. Crear el nuevo secreto
echo "NuevaContraseña2026!" | docker secret create db_password_v2 -

# 2. Actualizar el servicio para usar el nuevo secreto
docker service update \
  --secret-rm db_password \
  --secret-add source=db_password_v2,target=db_password \
  api

# 3. Eliminar el secreto viejo
docker secret rm db_password
```

### 8.4.4 Configs: configuración no sensible

Los **configs** son similares a los secrets, PERO NO ESTÁN ENCRIPTADOS. Son para archivos de configuración no sensibles (nginx.conf, php.ini, etc.).

```bash
# Crear un config desde un archivo
docker config create nginx_config ./nginx.conf

# Usar en un servicio
docker service create \
  --name web \
  --config source=nginx_config,target=/etc/nginx/nginx.conf \
  nginx

# Listar
docker config ls

# Inspeccionar (muestra los datos, no está encriptado)
docker config inspect --pretty nginx_config
```

**Diferencia entre Secrets y Configs**:

| Característica | Secrets | Configs |
|---------------|---------|---------|
| **Encriptado** | Sí (AES-256-GCM) | No |
| **Tamaño máximo** | 500 KB | 500 KB |
| **Almacenamiento** | Raft log encriptado | Raft log sin encriptar |
| **Montaje** | tmpfs (RAM, no disco) | tmpfs (RAM, no disco) |
| **Uso típico** | Contraseñas, tokens, claves API | nginx.conf, php.ini, etc. |
| **Visible en inspect** | No | Sí |

### 8.4.5 Routing Mesh: balanceo de carga integrado

El **Routing Mesh** es una de las características más potentes de Swarm. Funciona así:

**Cualquier nodo del cluster** (manager o worker) puede recibir tráfico para cualquier servicio publicado, incluso si ese nodo no tiene ninguna réplica de ese servicio. El nodo reenvía automáticamente el tráfico a un nodo que SÍ tenga la réplica.

```
┌─────────────────────────────────────────────────────────┐
│                   ROUTING MESH                          │
│                                                         │
│  Cliente ──► http://192.168.1.10:8080                  │
│  (nodo sin réplica)                                     │
│                  │                                      │
│                  ▼                                      │
│  ┌──────────────────────────────┐                       │
│  │        Nodo 1 (manager)      │                       │
│  │     IP: 192.168.1.10         │                       │
│  │                              │                       │
│  │  ┌──────────────────────┐    │                       │
│  │  │   IPVS Load Balancer │    │                       │
│  │  │   (kernel-level)     │────┼──► ¿Dónde está web?   │
│  │  └──────────────────────┘    │                       │
│  │                              │                       │
│  │  No tiene réplica de web     │                       │
│  └──────────────────────────────┘                       │
│                                                         │
│  ┌──────────────────────────────┐                       │
│  │        Nodo 2 (worker)       │                       │
│  │     IP: 192.168.1.20         │                       │
│  │                              │                       │
│  │  ┌──────────┐                │                       │
│  │  │ web:1    │ ◄──────────────┼─── Tráfico reenviado  │
│  │  │ :80      │                │                       │
│  │  └──────────┘                │                       │
│  └──────────────────────────────┘                       │
│                                                         │
│  El cliente recibe respuesta sin saber                  │
│  que el tráfico fue redirigido.                         │
└─────────────────────────────────────────────────────────┘
```

**Implicaciones importantes del Routing Mesh**:

1. **Puedes poner un load balancer externo frente a cualquier subconjunto de nodos**, no necesitas apuntar al nodo específico donde corre el servicio.
2. **Los puertos publicados están abiertos en TODOS los nodos**, incluso los que no tienen réplicas del servicio.
3. **IPVS** (IP Virtual Server) en el kernel de Linux maneja el balanceo a nivel 4 (TCP/UDP).

**Modos de publicación de puertos**:

```bash
# Modo "ingress" (por defecto): routing mesh completo
docker service create --publish published=8080,target=80,mode=ingress ...

# Modo "host": solo el puerto en el nodo donde corre la réplica
docker service create --publish published=8080,target=80,mode=host ...
```

Con `mode=host`, pierdes el routing mesh pero ganas performance (sin el salto extra de red). Útil para servicios de alta demanda donde controlas el balanceo externamente.

---


## 8.5 Kubernetes — PARTE 1: ARQUITECTURA

Kubernetes (K8s) es un sistema open-source creado por Google en 2014, basado en 15 años de experiencia corriendo contenedores a escala planetaria (Borg y Omega). Hoy es mantenido por la Cloud Native Computing Foundation (CNCF) y es el estándar indiscutible de la orquestación.

### 8.5.1 Componentes del Control Plane

El **Control Plane** es el cerebro de Kubernetes. Toma decisiones globales sobre el cluster y responde a eventos.

```
┌──────────────────────────────────────────────────────────────────┐
│                       CONTROL PLANE                              │
│                                                                  │
│  ┌──────────────────┐   ┌──────────────────┐                    │
│  │   API Server     │   │      etcd        │                    │
│  │   (kube-apiserver)│  │  (Base de datos) │                    │
│  │                  │   │                  │                    │
│  │ • REST API       │◄─►│ • Clave-Valor    │                    │
│  │ • Validación     │   │ • Consistente    │                    │
│  │ • Autenticación  │   │ • Raft-based     │                    │
│  │ • Autorización   │   │ • Todo el estado │                    │
│  └────────┬─────────┘   └──────────────────┘                    │
│           │                                                      │
│           │ watch de cambios                                     │
│           │                                                      │
│  ┌────────▼─────────┐   ┌───────────────────────────┐           │
│  │    Scheduler     │   │   Controller Manager       │           │
│  │(kube-scheduler)  │   │ (kube-controller-manager)  │           │
│  │                  │   │                            │           │
│  │ • Asigna Pods    │   │ • Node Controller          │           │
│  │   a Nodos        │   │ • ReplicaSet Controller    │           │
│  │ • Afinidad       │   │ • Deployment Controller    │           │
│  │ • Recursos       │   │ • Service Controller       │           │
│  └──────────────────┘   │ • Y muchos más...          │           │
│                          └───────────────────────────┘           │
│                                                                  │
│  ┌──────────────────────────────────────┐                        │
│  │   Cloud Controller Manager           │ (opcional, en cloud)  │
│  │   (cloud-controller-manager)         │                        │
│  │                                      │                        │
│  │   • Node Controller (cloud)          │                        │
│  │   • Route Controller                 │                        │
│  │   • Service Controller (LB)          │                        │
│  └──────────────────────────────────────┘                        │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

**API Server (`kube-apiserver`)**:
Es el frontend único del Control Plane. Toda comunicación (interna y externa) pasa por aquí. Es RESTful. Valida y procesa las peticiones, luego escribe los objetos en etcd.

**etcd**:
Base de datos clave-valor distribuida y consistente (Raft). Aquí se almacena TODO el estado del cluster: qué pods existen, qué nodos hay, configmaps, secrets, TODO. Si etcd muere, Kubernetes queda ciego.

**Scheduler (`kube-scheduler`)**:
Observa los Pods recién creados que no tienen un nodo asignado. Evalúa qué nodo es el mejor basándose en recursos disponibles, afinidad/anti-afinidad, taints/tolerations, etc. Asigna el Pod a un nodo específico.

**Controller Manager (`kube-controller-manager`)**:
Ejecuta múltiples controladores en un solo binario. Cada controlador es un bucle de reconciliación que observa el estado actual y lo compara con el estado deseado. Si no coinciden, toma acciones.

```
BUCLE DE RECONCILIACIÓN (Controller Pattern):

    ┌──────────────────────────────────────┐
    │                                      │
    │   ┌──────────────────────────┐       │
    │   │  ESTADO DESEADO          │       │
    │   │  (Deployment: 5 réplicas)│       │
    │   └────────────┬─────────────┘       │
    │                │                     │
    │        ¿Coincide? ──── Sí ───► Nada │
    │                │                     │
    │              No │                    │
    │                ▼                     │
    │   ┌──────────────────────────┐       │
    │   │  TOMAR ACCIÓN            │       │
    │   │  (Crear Pod, Eliminar    │       │
    │   │   Pod, Actualizar...)    │       │
    │   └──────────────────────────┘       │
    │                                      │
    └──────────────────────────────────────┘
    ◄──── Se repite continuamente ──────────
```

### 8.5.2 Componentes de los Worker Nodes

```
┌──────────────────────────────────────────────────────────┐
│                     WORKER NODE                           │
│                                                          │
│  ┌──────────────────────────────────────────────────┐    │
│  │                   kubelet                        │    │
│  │                                                  │    │
│  │  • Agente que corre en cada nodo                 │    │
│  │  • Recibe PodSpecs del API Server                │    │
│  │  • Asegura que los contenedores están corriendo  │    │
│  │  • Reporta estado del nodo y pods                │    │
│  │  • No maneja contenedores no creados por K8s     │    │
│  └──────────────────────────────────────────────────┘    │
│                                                          │
│  ┌──────────────────────────────────────────────────┐    │
│  │                 kube-proxy                        │    │
│  │                                                  │    │
│  │  • Proxy de red que corre en cada nodo           │    │
│  │  • Mantiene reglas de red (iptables/IPVS)        │    │
│  │  • Implementa Services: balancea tráfico a Pods  │    │
│  │  • Permite comunicación Pod-a-Pod y Pod-a-externo│    │
│  └──────────────────────────────────────────────────┘    │
│                                                          │
│  ┌──────────────────────────────────────────────────┐    │
│  │          Container Runtime                       │    │
│  │                                                  │    │
│  │  • containerd (default desde K8s 1.24)           │    │
│  │  • CRI-O                                          │    │
│  │  • Docker (deprecated, eliminado en 1.24+)       │    │
│  │                                                  │    │
│  │  • Interface: CRI (Container Runtime Interface)  │    │
│  └──────────────────────────────────────────────────┘    │
│                                                          │
│  ┌────────┐  ┌────────┐  ┌────────┐  ┌────────┐        │
│  │ Pod A  │  │ Pod B  │  │ Pod C  │  │ Pod D  │        │
│  │┌──────┐│  │┌──────┐│  │┌──────┐│  │┌──────┐│        │
│  ││Cont.1││  ││Cont.1││  ││Cont.1││  ││Cont.1││        │
│  │└──────┘│  ││┌─────┐│  │└──────┘│  │└──────┘│        │
│  │        │  │││Cont ││  │        │  │        │        │
│  │        │  │││  2  ││  │        │  │        │        │
│  │        │  ││└─────┘│  │        │  │        │        │
│  └────────┘  └────────┘  └────────┘  └────────┘        │
│                                                          │
└──────────────────────────────────────────────────────────┘
```

**kubelet**:
Es el "capataz" del nodo. No fue creado por Kubernetes (preexiste). Toma un conjunto de PodSpecs y se asegura de que los contenedores descritos estén corriendo y saludables. Reporta al API Server.

**kube-proxy**:
Implementa el concepto de **Service** de Kubernetes. Mantiene las reglas de red que permiten el balanceo de carga entre los Pods de un Service. Puede usar iptables o IPVS.

**Container Runtime**:
El software que realmente ejecuta los contenedores. Kubernetes soporta cualquier runtime que implemente la CRI (Container Runtime Interface): containerd, CRI-O, etc.

### 8.5.3 Flujo de una petición completa

Vamos a seguir el viaje de un simple `kubectl apply -f deployment.yaml` paso a paso:

```
╔═══════════════════════════════════════════════════════════════╗
║ PASO 1: El usuario ejecuta                                   ║
║   $ kubectl apply -f deployment.yaml                         ║
╚══════════════════╤════════════════════════════════════════════╝
                   │
                   ▼
     kubectl convierte el YAML a JSON
     y lo envía por HTTPS al API Server
                   │
                   ▼
╔═══════════════════════════════════════════════════════════════╗
║ PASO 2: API Server recibe la petición                        ║
║   • Autenticación (¿quién eres?)                             ║
║   • Autorización (¿tienes permiso?)                          ║
║   • Admisión (¿cumple políticas?)                            ║
║   • Validación (¿es válido el objeto?)                       ║
║   • Escribe el Deployment en etcd                            ║
╚══════════════════╤════════════════════════════════════════════╝
                   │
                   ▼
╔═══════════════════════════════════════════════════════════════╗
║ PASO 3: Deployment Controller detecta el cambio              ║
║   (está watcheando el API Server)                            ║
║   • Estado actual: no hay ReplicaSet                         ║
║   • Estado deseado: Deployment con 3 réplicas                ║
║   • Acción: crea un ReplicaSet                               ║
║   • Escribe el ReplicaSet en API Server → etcd               ║
╚══════════════════╤════════════════════════════════════════════╝
                   │
                   ▼
╔═══════════════════════════════════════════════════════════════╗
║ PASO 4: ReplicaSet Controller detecta el cambio              ║
║   • Estado actual: no hay Pods                               ║
║   • Estado deseado: 3 Pods                                   ║
║   • Acción: crea 3 Pods (sin nodo asignado)                  ║
║   • Escribe los Pods en API Server → etcd                    ║
╚══════════════════╤════════════════════════════════════════════╝
                   │
                   ▼
╔═══════════════════════════════════════════════════════════════╗
║ PASO 5: Scheduler detecta Pods sin nodo                      ║
║   • Evalúa cada nodo (recursos, afinidad, taints)            ║
║   • Asigna cada Pod al mejor nodo                            ║
║   • Actualiza Pod.spec.nodeName en API Server → etcd        ║
╚══════════════════╤════════════════════════════════════════════╝
                   │
                   ▼
╔═══════════════════════════════════════════════════════════════╗
║ PASO 6: kubelet detecta Pods asignados a su nodo             ║
║   • Lee el PodSpec del API Server                            ║
║   • Llama al Container Runtime (via CRI)                     ║
║   • El runtime descarga la imagen, crea el contenedor        ║
║   • kubelet reporta estado Running al API Server             ║
╚═══════════════════════════════════════════════════════════════╝

ESTADO FINAL: 3 Pods corriendo en 3 nodos.
Todo desde un solo comando. Todo declarativo.
```

**Tiempos típicos**:
- Paso 1-2 (API Server + etcd): milisegundos.
- Paso 3-4 (Controllers): segundos (depende de la complejidad).
- Paso 5 (Scheduler): milisegundos.
- Paso 6 (kubelet + image pull): segundos a minutos (depende del tamaño de la imagen).

### 8.5.4 Namespaces: entornos lógicos

Kubernetes usa **Namespaces** como mecanismo de aislamiento lógico dentro de un mismo cluster físico. Son como "folders" para tus recursos.

```
┌─────────────────────────────────────────────────────────┐
│               CLUSTER KUBERNETES                        │
│                                                         │
│  ┌─────────────────┐ ┌─────────────────┐               │
│  │   Namespace     │ │   Namespace     │               │
│  │   "dev"         │ │   "production"  │               │
│  │                 │ │                 │               │
│  │ ┌──────┐        │ │ ┌──────┐        │               │
│  │ │ web  │        │ │ │ web  │        │               │
│  │ └──────┘        │ │ └──────┘        │               │
│  │ ┌──────┐        │ │ ┌──────┐        │               │
│  │ │ api  │        │ │ │ api  │        │               │
│  │ └──────┘        │ │ └──────┘        │               │
│  │ ┌──────┐        │ │ ┌──────┐        │               │
│  │ │ db   │        │ │ │ db   │        │               │
│  │ └──────┘        │ │ └──────┘        │               │
│  └─────────────────┘ └─────────────────┘               │
│                                                         │
│  ┌─────────────────────────────────────┐                │
│  │   Namespace "kube-system"           │                │
│  │   (componentes del propio K8s)      │                │
│  │   - kube-dns, metrics-server, etc.  │                │
│  └─────────────────────────────────────┘                │
│                                                         │
│  ┌─────────────────────────────────────┐                │
│  │   Namespace "default"               │                │
│  │   (si no especificas namespace)     │                │
│  └─────────────────────────────────────┘                │
└─────────────────────────────────────────────────────────┘
```

Los namespaces son puramente lógicos: los Pods de diferentes namespaces pueden estar en el mismo nodo físico. No hay aislamiento de recursos por defecto (aunque puedes configurar ResourceQuotas y NetworkPolicies por namespace).

```bash
# Listar namespaces
kubectl get namespaces

# Salida típica:
# NAME              STATUS   AGE
# default           Active   30d
# kube-system       Active   30d
# kube-public       Active   30d
# kube-node-lease   Active   30d
```

---

## 8.6 Kubernetes — PARTE 2: OBJETOS FUNDAMENTALES

Kubernetes es una plataforma extensible. Pero todo se construye sobre un conjunto de objetos fundamentales. Vamos a recorrerlos uno por uno.

### 8.6.1 Pods: la unidad mínima

Un **Pod** es la unidad más pequeña y básica que puedes desplegar en Kubernetes. Es un grupo de uno o más contenedores que:

- **Comparten la misma IP** y puertos de red (localhost entre ellos).
- **Comparten el mismo volumen** (pueden montar los mismos discos).
- **Se ejecutan en el mismo nodo** (siempre).
- **Se crean y destruyen juntos** (ciclo de vida compartido).

```
┌──────────────────────────────────────────────┐
│                   POD                        │
│                                              │
│  IP: 10.244.1.5                              │
│  Namespace: default                          │
│  Nodo: worker-01                             │
│                                              │
│  ┌────────────────────┐                      │
│  │  Contenedor: web   │                      │
│  │  Image: nginx:1.25 │                      │
│  │  Port: 80          │                      │
│  └────────────────────┘                      │
│                                              │
│  ┌────────────────────┐                      │
│  │  Contenedor:       │                      │
│  │  log-shipper       │  ← Sidecar          │
│  │  Image: fluentd    │                      │
│  └────────────────────┘                      │
│                                              │
│  ┌────────────────────────────────────┐      │
│  │  Shared Volume: /var/log           │      │
│  │  (ambos contenedores pueden leer   │      │
│  │   y escribir aquí)                 │      │
│  └────────────────────────────────────┘      │
│                                              │
└──────────────────────────────────────────────┘
```

**Patrones de Pods**:

1. **Sidecar**: un contenedor auxiliar que extiende la funcionalidad del principal (ej. log shipper, proxy).
2. **Ambassador**: proxy que abstrae la conexión a un servicio externo.
3. **Adapter**: normaliza la salida del contenedor principal (métricas, logs).
4. **Init Container**: contenedor que se ejecuta antes del principal (migraciones de BD, esperar dependencias).

**Init Containers en acción**:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: web-with-init
spec:
  initContainers:
    - name: wait-for-db
      image: busybox:1.36
      command:
        - 'sh'
        - '-c'
        - |
          echo "Esperando que la base de datos esté lista..."
          until nc -z db-service 5432; do
            echo "DB no lista aún, reintentando en 2s..."
            sleep 2
          done
          echo "Base de datos lista!"
    - name: migrate-db
      image: api:latest
      command: ['npm', 'run', 'migrate']
  containers:
    - name: web
      image: nginx:1.25
```

Los init containers se ejecutan **secuencialmente** (primero `wait-for-db`, luego `migrate-db`). Solo cuando todos terminan exitosamente, arranca el contenedor principal `web`.

### 8.6.2 Deployments: gestión declarativa de Pods

Un **Deployment** es el mecanismo estándar para gestionar Pods de forma declarativa. Define:

- Qué imagen correr.
- Cuántas réplicas (Pods) mantener.
- Estrategia de actualización (rolling, recreate).
- Labels y selectores.

```yaml
# deployment-web.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
  namespace: default
  labels:
    app: web
    tier: frontend
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1           # Cuántos Pods extra se pueden crear durante update
      maxUnavailable: 0     # Cuántos Pods pueden estar no disponibles (0 = sin downtime)
  template:
    metadata:
      labels:
        app: web
        version: v1
    spec:
      containers:
        - name: nginx
          image: nginx:1.25
          ports:
            - containerPort: 80
              protocol: TCP
          resources:
            requests:
              cpu: "250m"        # 0.25 cores
              memory: "256Mi"
            limits:
              cpu: "500m"        # 0.5 cores
              memory: "512Mi"
          livenessProbe:
            httpGet:
              path: /
              port: 80
            initialDelaySeconds: 10
            periodSeconds: 5
          readinessProbe:
            httpGet:
              path: /health
              port: 80
            initialDelaySeconds: 5
            periodSeconds: 3
          env:
            - name: ENVIRONMENT
              value: "production"
            - name: VERSION
              value: "v1.25"
```

**Estrategias de Deployment**:

| Estrategia | Descripción |
|-----------|-------------|
| **RollingUpdate** (default) | Reemplazo gradual, sin downtime. Controlado por maxSurge y maxUnavailable. |
| **Recreate** | Destruye TODOS los Pods y recrea. Hay downtime. Simple. |
| **Azul-Verde** | Manual con Services: despliegas la nueva versión, verificas, cambias el Service. |
| **Canary** | Con Ingress + múltiples Deployments: % de tráfico a la nueva versión. |

**Rolling Update en acción**:

```
Despliegue de v1 a v2 con 3 réplicas:

          t=0s      t=10s      t=20s      t=30s
Pod 1:  [v1]  →                  →        [v2]
Pod 2:  [v1]  →    [v2]  →     [v2]  →   [v2]
Pod 3:  [v1]  →    [v1]  →                →   [v2]
Extra:                  [v2]   (maxSurge=1 permite 4 pods temporalmente)

Total Pods durante update: 3-4 (nunca menos de 3 con maxUnavailable=0)
```

### 8.6.3 Services: IP fija y DNS para Pods efímeros

Los Pods son efímeros. Se crean, se destruyen, cambian de IP. Un **Service** proporciona una IP fija (ClusterIP) y un nombre DNS estable que apunta a un conjunto dinámico de Pods.

```
┌────────────────────────────────────────────────────────┐
│                    SERVICE "web"                       │
│                                                        │
│  ClusterIP: 10.96.0.100                                │
│  DNS: web.default.svc.cluster.local                    │
│  Puerto: 80 → targetPort: 80                          │
│  Selector: app=web                                     │
│                                                        │
│  ┌───────────┐   ┌───────────┐   ┌───────────┐       │
│  │  Pod web  │   │  Pod web  │   │  Pod web  │       │
│  │ 10.244.1.5│   │ 10.244.2.3│   │ 10.244.3.7│       │
│  │  Running  │   │  Running  │   │  Running  │       │
│  └───────────┘   └───────────┘   └───────────┘       │
│                                                        │
│  Los Pods van y vienen.                                │
│  El Service permanece.                                 │
│  El tráfico a 10.96.0.100:80 se balancea               │
│  entre los 3 Pods.                                     │
└────────────────────────────────────────────────────────┘
```

**Tipos de Service**:

```yaml
# 1. ClusterIP (default): solo accesible dentro del cluster
apiVersion: v1
kind: Service
metadata:
  name: web
spec:
  type: ClusterIP
  selector:
    app: web
  ports:
    - port: 80        # Puerto del Service
      targetPort: 80  # Puerto del contenedor
      protocol: TCP

---
# 2. NodePort: expone en un puerto de cada nodo (30000-32767)
apiVersion: v1
kind: Service
metadata:
  name: web-nodeport
spec:
  type: NodePort
  selector:
    app: web
  ports:
    - port: 80
      targetPort: 80
      nodePort: 31000    # Puerto fijo en cada nodo (opcional, si no, auto-asignado)
      protocol: TCP

---
# 3. LoadBalancer: balanceador externo (cloud provider)
apiVersion: v1
kind: Service
metadata:
  name: web-lb
spec:
  type: LoadBalancer
  selector:
    app: web
  ports:
    - port: 80
      targetPort: 80
      protocol: TCP
  # En cloud, esto provisiona un ELB/ALB/Load Balancer real

---
# 4. ExternalName: redirige a un DNS externo (no usa selectores)
apiVersion: v1
kind: Service
metadata:
  name: external-db
spec:
  type: ExternalName
  externalName: db.example.com  # DNS externo
```

**Resumen de tipos**:

| Tipo | Accesible desde | IP | Puerto | Caso de uso |
|------|----------------|----|--------|-------------|
| **ClusterIP** | Dentro del cluster | 10.x.x.x | cualquiera | Comunicación interna |
| **NodePort** | Fuera del cluster (IP:Port de cualquier nodo) | Nodo | 30000-32767 | Dev/test, demos |
| **LoadBalancer** | Fuera del cluster | Externa (cloud LB) | cualquiera | Producción en cloud |
| **ExternalName** | Dentro del cluster | DNS CNAME | N/A | Apuntar a servicio externo |

### 8.6.4 Ingress: reglas de ruteo HTTP/HTTPS

Un **Ingress** gestiona el acceso externo a los Services dentro del cluster, típicamente HTTP/HTTPS. Proporciona:

- **Ruteo por hostname** (virtual hosting).
- **Ruteo por path** (URL-based routing).
- **Terminación TLS** (HTTPS).
- **Balanceo de carga**.

```
                     ┌──────────────────────┐
                     │    CLIENTE EXTERNO    │
                     └──────────┬───────────┘
                                │
                https://api.example.com/v1/users
                https://example.com/
                                │
                                ▼
                     ┌──────────────────────┐
                     │   LOAD BALANCER       │
                     │   (Cloud o MetalLB)   │
                     └──────────┬───────────┘
                                │
                                ▼
          ┌─────────────────────────────────────────┐
          │            INGRESS CONTROLLER           │
          │          (NGINX, Traefik, HAProxy...)   │
          │                                         │
          │  ┌───────────────────────────────────┐  │
          │  │ Reglas de Ingress:                │  │
          │  │                                   │  │
          │  │ api.example.com/v1/* → api-service│  │
          │  │ example.com/*         → web-service│  │
          │  │ *.example.com         → TLS cert   │  │
          │  └───────────────────────────────────┘  │
          └───────┬──────────────────┬──────────────┘
                  │                  │
                  ▼                  ▼
    ┌──────────────────┐  ┌──────────────────┐
    │  Service: web    │  │  Service: api    │
    │  ClusterIP:      │  │  ClusterIP:      │
    │  10.96.0.100:80  │  │  10.96.0.101:3000│
    └──────────────────┘  └──────────────────┘
```

**Manifiesto de Ingress**:

```yaml
# ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: main-ingress
  namespace: default
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - example.com
        - api.example.com
      secretName: tls-certificate
  rules:
    - host: example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: web
                port:
                  number: 80
          - path: /static
            pathType: Prefix
            backend:
              service:
                name: static-files
                port:
                  number: 80
    - host: api.example.com
      http:
        paths:
          - path: /v1
            pathType: Prefix
            backend:
              service:
                name: api-v1
                port:
                  number: 3000
          - path: /v2
            pathType: Prefix
            backend:
              service:
                name: api-v2
                port:
                  number: 3000
```

**IMPORTANTE**: Un Ingress necesita un **Ingress Controller** corriendo en el cluster. NGINX Ingress Controller es el más popular:

```bash
# Instalar NGINX Ingress Controller
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.9.0/deploy/static/provider/cloud/deploy.yaml

# Verificar que está corriendo
kubectl get pods -n ingress-nginx
```

### 8.6.5 ConfigMaps: configuraciones no sensibles

```yaml
# configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: default
data:
  # Propiedades clave-valor simples
  APP_ENV: "production"
  LOG_LEVEL: "info"
  API_TIMEOUT: "30s"

  # Archivos completos
  nginx.conf: |
    server {
        listen 80;
        server_name example.com;
        location / {
            proxy_pass http://api:3000;
            proxy_set_header Host $host;
        }
    }

  allowed_origins.txt: |
    https://example.com
    https://app.example.com
    https://admin.example.com
```

**Usar ConfigMap**:

```yaml
# deployment-using-configmap.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api
spec:
  replicas: 3
  selector:
    matchLabels:
      app: api
  template:
    spec:
      containers:
        - name: api
          image: api:latest
          # Opción A: Como variables de entorno
          envFrom:
            - configMapRef:
                name: app-config
          # Opción B: Variables individuales desde el ConfigMap
          env:
            - name: LOG_LEVEL
              valueFrom:
                configMapKeyRef:
                  name: app-config
                  key: LOG_LEVEL
          # Opción C: Como archivo montado
          volumeMounts:
            - name: nginx-config
              mountPath: /etc/nginx/nginx.conf
              subPath: nginx.conf
      volumes:
        - name: nginx-config
          configMap:
            name: app-config
            items:
              - key: nginx.conf
                path: nginx.conf
```

### 8.6.6 Secrets: datos sensibles

```yaml
# secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
  namespace: default
type: Opaque
data:
  # Valores en base64 (NO encriptados, solo codificados)
  username: YXBwdXNlcg==        # echo -n "appuser" | base64
  password: U3VwZXJTZWNyZXQxMjM= # echo -n "SuperSecret123" | base64
```

O usando `stringData` (Kubernetes codifica a base64 automáticamente):

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
stringData:
  username: appuser
  password: SuperSecret123
```

**¡ADVERTENCIA IMPORTANTE!**: Los Secrets en Kubernetes NO están encriptados por defecto. Solo están codificados en base64. Cualquiera con acceso a etcd o al API Server puede leerlos. Para encriptación real en etcd, debes configurar **EncryptionConfiguration**:

```yaml
# encryption-config.yaml (se configura a nivel de API Server)
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
  - resources:
      - secrets
    providers:
      - aescbc:
          keys:
            - name: key1
              secret: c2VjcmV0LWtleS1mb3ItYWVzLWNiYy1lbmNyeXB0aW9u
      - identity: {}
```

**Usar Secrets en un Deployment**:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api
spec:
  replicas: 3
  selector:
    matchLabels:
      app: api
  template:
    spec:
      containers:
        - name: api
          image: api:latest
          env:
            # Como variable de entorno (no recomendado para secretos)
            - name: DB_USERNAME
              valueFrom:
                secretKeyRef:
                  name: db-credentials
                  key: username
            # Ruta al archivo (recomendado)
            - name: DB_PASSWORD_FILE
              value: /etc/secrets/db/password
          volumeMounts:
            - name: db-secrets
              mountPath: /etc/secrets/db
              readOnly: true
      volumes:
        - name: db-secrets
          secret:
            secretName: db-credentials
            items:
              - key: password
                path: password     # /etc/secrets/db/password
              - key: username
                path: username
            defaultMode: 0400      # Solo lectura para el owner
```

### 8.6.7 PersistentVolumes (PV) y PersistentVolumeClaims (PVC)

Kubernetes desacopla el almacenamiento del Pod. Un administrador crea **PersistentVolumes** (la capacidad física). Un desarrollador solicita almacenamiento con **PersistentVolumeClaims** (la demanda).

```
┌─────────────────────────────────────────────────────────┐
│              MODELO DE ALMACENAMIENTO                    │
│                                                         │
│  ┌───────────────────┐    ┌───────────────────┐         │
│  │ PersistentVolume  │    │ PersistentVolume  │         │
│  │ (PV)              │    │ (PV)              │         │
│  │                   │    │                   │         │
│  │ Capacidad: 10Gi   │    │ Capacidad: 100Gi  │         │
│  │ Tipo: NFS         │    │ Tipo: SSD (cloud) │         │
│  │ Path: /exports/pv1│    │ ID: vol-abc123    │         │
│  └────────┬──────────┘    └────────┬──────────┘         │
│           │                        │                     │
│           └──────────┬─────────────┘                     │
│                      │                                   │
│                      ▼                                   │
│           ┌───────────────────┐                          │
│           │PersistentVolume   │                          │
│           │Claim (PVC)        │                          │
│           │                   │                          │
│           │ Storage: 5Gi      │                          │
│           │ Access: ReadWriteOnce                      │
│           └────────┬──────────┘                          │
│                    │                                     │
│                    ▼                                     │
│           ┌───────────────────┐                          │
│           │       Pod         │                          │
│           │                   │                          │
│           │ volumeMounts:     │                          │
│           │  - /var/lib/data  │                          │
│           └───────────────────┘                          │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

**Ejemplo PV**:

```yaml
# persistent-volume.yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: db-pv
spec:
  capacity:
    storage: 10Gi
  volumeMode: Filesystem
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: standard
  hostPath:
    path: /data/db
  # En cloud usarías:
  # awsElasticBlockStore:
  #   volumeID: vol-abc123
  #   fsType: ext4
```

**Ejemplo PVC**:

```yaml
# persistent-volume-claim.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: db-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
  storageClassName: standard
```

**Usar PVC en un Deployment**:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: db
spec:
  replicas: 1
  selector:
    matchLabels:
      app: db
  template:
    spec:
      containers:
        - name: postgres
          image: postgres:16
          volumeMounts:
            - name: db-storage
              mountPath: /var/lib/postgresql/data
      volumes:
        - name: db-storage
          persistentVolumeClaim:
            claimName: db-pvc
```

**StorageClasses**: Permiten provisionamiento dinámico. En lugar de crear PVs manualmente, el PVC los crea automáticamente:

```yaml
# storage-class.yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-ssd
provisioner: kubernetes.io/aws-ebs   # Provisionador del cloud
parameters:
  type: gp3
  iops: "3000"
reclaimPolicy: Delete                # Elimina el PV cuando el PVC se elimina
volumeBindingMode: WaitForFirstConsumer
```

### 8.6.8 Namespaces en Kubernetes

Ya los mencionamos, pero es importante conocer los comandos:

```bash
# Crear namespace
kubectl create namespace dev
kubectl create namespace staging
kubectl create namespace production

# Trabajar en un namespace específico (sin tener que poner -n cada vez)
kubectl config set-context --current --namespace=dev

# Herramientas útiles (instalar por separado)
# kubens: cambia el namespace activo
kubens production

# kubectx: cambia entre clusters (contextos)
kubectx minikube
kubectx production-cluster
```

---

## 8.7 Kubernetes — PARTE 3: MANOS A LA OBRA

### 8.7.1 Opciones para cluster local

Para desarrollo y aprendizaje, tienes varias opciones:

| Herramienta | Características | Mejor para |
|-------------|-----------------|------------|
| **Minikube** | Un solo nodo, soporta addons | Aprendizaje, desarrollo individual |
| **Kind** (Kubernetes IN Docker) | Nodos como contenedores Docker, multi-nodo | CI/CD, testing |
| **k3s** | Kubernetes ligero de Rancher, IoT/edge | Producción ligera, Raspberry Pi |
| **MicroK8s** | Snap de Canonical, multi-nodo | Desarrollo en Ubuntu |

### 8.7.2 Minikube: el clásico

```bash
# Instalar Minikube (macOS)
brew install minikube

# Iniciar un cluster (con Docker como driver)
minikube start --driver=docker --cpus=4 --memory=8192

# Verificar
minikube status
kubectl get nodes
```

Salida:

```
NAME       STATUS   ROLES           AGE   VERSION
minikube   Ready    control-plane   2m    v1.29.0
```

```bash
# Ver addons disponibles
minikube addons list

# Habilitar dashboard e ingress
minikube addons enable dashboard
minikube addons enable ingress

# Abrir el dashboard
minikube dashboard
```

```bash
# Obtener la IP del cluster
minikube ip

# Si estás en macOS y usas Docker driver:
# minikube service <service-name> --url  # Para acceder a un servicio NodePort/LB
```

### 8.7.3 Kind: Kubernetes en Docker

```bash
# Instalar Kind
brew install kind

# Crear un cluster de 3 nodos (1 control-plane + 2 workers)
kind create cluster --name learning-k8s --config - <<EOF
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
  - role: control-plane
  - role: worker
  - role: worker
EOF

# Verificar
kubectl cluster-info --context kind-learning-k8s
kubectl get nodes
```

Salida:

```
NAME                          STATUS   ROLES           AGE   VERSION
learning-k8s-control-plane    Ready    control-plane   1m    v1.29.0
learning-k8s-worker           Ready    <none>          1m    v1.29.0
learning-k8s-worker2          Ready    <none>          1m    v1.29.0
```

### 8.7.4 Comandos fundamentales de kubectl

```bash
# ──── Crear recursos ────

# Desde línea de comandos (imperativo)
kubectl run nginx --image=nginx:1.25 --port=80
kubectl create deployment api --image=api:latest --replicas=3
kubectl expose deployment api --port=3000 --target-port=3000 --type=NodePort

# Desde archivo YAML (declarativo) — RECOMENDADO
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml
kubectl apply -f configmap.yaml
kubectl apply -f secret.yaml
kubectl apply -f ingress.yaml

# Aplicar todo un directorio
kubectl apply -f ./k8s/

# Aplicar desde URL
kubectl apply -f https://example.com/deployment.yaml

# ──── Ver recursos ────

kubectl get pods                    # Listar Pods
kubectl get pods -o wide            # Con más detalles (IP, nodo)
kubectl get pods --show-labels      # Mostrar labels

kubectl get deployments
kubectl get services
kubectl get replicasets
kubectl get configmaps
kubectl get secrets
kubectl get pv,pvc
kubectl get ingress
kubectl get all                     # TODO (pods, services, deployments, replicasets)

# ──── Inspeccionar recursos ────

kubectl describe pod <pod-name>              # Descripción detallada
kubectl describe deployment <deployment-name>
kubectl describe service <service-name>

kubectl logs <pod-name>                      # Logs de un Pod
kubectl logs <pod-name> -c <container-name>  # Logs de un contenedor específico
kubectl logs -f <pod-name>                   # Seguir logs (tail -f)
kubectl logs --tail=50 <pod-name>            # Últimas 50 líneas
kubectl logs -l app=web --all-containers     # Logs de todos los Pods con label app=web

kubectl exec -it <pod-name> -- /bin/bash     # Shell dentro del Pod
kubectl exec <pod-name> -- ls /app           # Ejecutar comando
kubectl exec <pod-name> -c <container> -- sh # Shell en contenedor específico

kubectl port-forward <pod-name> 8080:80      # Forward de puerto local al Pod
kubectl port-forward svc/web 8080:80         # Forward a un Service

# ──── Etiquetas y anotaciones ────

kubectl label pod <pod-name> env=production
kubectl label node <node-name> disk=ssd
kubectl annotate deployment web description="Frontend web app"

# ──── Escalar ────

kubectl scale deployment web --replicas=5
kubectl scale deployment api --replicas=0    # Escalar a 0 (apagar todo)

# ──── Actualizaciones ────

kubectl set image deployment/web nginx=nginx:1.26
kubectl edit deployment web                  # Editar YAML en vivo
kubectl patch deployment web -p '{"spec":{"replicas":10}}'

# ──── Rollouts ────

kubectl rollout status deployment/web        # Ver estado del rollout
kubectl rollout history deployment/web       # Historial de revisiones
kubectl rollout history deployment/web --revision=3
kubectl rollout undo deployment/web          # Rollback a la versión anterior
kubectl rollout undo deployment/web --to-revision=2  # Rollback a revisión específica
kubectl rollout restart deployment/web       # Reiniciar todos los Pods

# ──── Eliminar recursos ────

kubectl delete pod <pod-name>                # Eliminar Pod (se recrea si es parte de un Deployment)
kubectl delete deployment web                # Eliminar Deployment
kubectl delete service web                   # Eliminar Service
kubectl delete -f deployment.yaml            # Eliminar desde archivo
kubectl delete all --all                     # Eliminar TODO en el namespace actual (¡PELIGRO!)

# ──── Operaciones avanzadas ────

kubectl top pods                             # Uso de recursos (requiere metrics-server)
kubectl top nodes
kubectl explain pod                          # Documentación del recurso
kubectl explain deployment.spec.selector     # Campo específico
kubectl api-resources                        # Todos los tipos de recursos disponibles
kubectl api-versions                         # Todas las versiones de API
```

### 8.7.5 Desplegar una aplicación completa

Vamos a desplegar una aplicación web completa paso a paso con YAML declarativo. Esta aplicación tiene 3 capas: web (Nginx), api (Node.js) y db (PostgreSQL). Cada componente tendrá su Deployment, Service, ConfigMap, Secrets y PersistentVolumeClaim.

**Paso 1: Crear el namespace**:

```yaml
# 01-namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: demo-app
```

**Paso 2: ConfigMap con variables de entorno**:

```yaml
# 02-configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: demo-app
data:
  APP_ENV: "production"
  LOG_LEVEL: "info"
  DB_HOST: "db-service"
  DB_PORT: "5432"
  DB_NAME: "appdb"
```

**Paso 3: Secrets**:

```yaml
# 03-secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-secrets
  namespace: demo-app
type: Opaque
stringData:
  DB_USER: "appuser"
  DB_PASSWORD: "SuperSecretPass2026!"
  JWT_SECRET: "my-256-bit-secret-key-for-jwt-tokens"
```

**Paso 4: PersistentVolumeClaim**:

```yaml
# 04-pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: db-pvc
  namespace: demo-app
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
  storageClassName: standard
```

**Paso 5: Deployment de la base de datos**:

```yaml
# 05-deployment-db.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: db
  namespace: demo-app
  labels:
    app: db
spec:
  replicas: 1
  selector:
    matchLabels:
      app: db
  template:
    metadata:
      labels:
        app: db
    spec:
      containers:
        - name: postgres
          image: postgres:16-alpine
          ports:
            - containerPort: 5432
          env:
            - name: POSTGRES_USER
              valueFrom:
                secretKeyRef:
                  name: app-secrets
                  key: DB_USER
            - name: POSTGRES_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: app-secrets
                  key: DB_PASSWORD
            - name: POSTGRES_DB
              valueFrom:
                configMapKeyRef:
                  name: app-config
                  key: DB_NAME
          resources:
            requests:
              cpu: "250m"
              memory: "256Mi"
            limits:
              cpu: "500m"
              memory: "512Mi"
          volumeMounts:
            - name: db-data
              mountPath: /var/lib/postgresql/data
          livenessProbe:
            exec:
              command:
                - pg_isready
                - -U
                - appuser
                - -d
                - appdb
            initialDelaySeconds: 30
            periodSeconds: 10
            timeoutSeconds: 5
          readinessProbe:
            exec:
              command:
                - pg_isready
                - -U
                - appuser
                - -d
                - appdb
            initialDelaySeconds: 5
            periodSeconds: 5
      volumes:
        - name: db-data
          persistentVolumeClaim:
            claimName: db-pvc
```

**Paso 6: Service de la base de datos**:

```yaml
# 06-service-db.yaml
apiVersion: v1
kind: Service
metadata:
  name: db-service
  namespace: demo-app
spec:
  type: ClusterIP
  selector:
    app: db
  ports:
    - port: 5432
      targetPort: 5432
      protocol: TCP
```

**Paso 7: Deployment de la API**:

```yaml
# 07-deployment-api.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api
  namespace: demo-app
  labels:
    app: api
spec:
  replicas: 3
  selector:
    matchLabels:
      app: api
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    metadata:
      labels:
        app: api
        version: v1
    spec:
      containers:
        - name: api
          image: api:latest
          ports:
            - containerPort: 3000
          env:
            - name: NODE_ENV
              valueFrom:
                configMapKeyRef:
                  name: app-config
                  key: APP_ENV
            - name: DB_HOST
              valueFrom:
                configMapKeyRef:
                  name: app-config
                  key: DB_HOST
            - name: DB_PORT
              valueFrom:
                configMapKeyRef:
                  name: app-config
                  key: DB_PORT
            - name: DB_USER
              valueFrom:
                secretKeyRef:
                  name: app-secrets
                  key: DB_USER
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: app-secrets
                  key: DB_PASSWORD
            - name: JWT_SECRET
              valueFrom:
                secretKeyRef:
                  name: app-secrets
                  key: JWT_SECRET
          resources:
            requests:
              cpu: "250m"
              memory: "256Mi"
            limits:
              cpu: "500m"
              memory: "512Mi"
          livenessProbe:
            httpGet:
              path: /health
              port: 3000
            initialDelaySeconds: 15
            periodSeconds: 20
          readinessProbe:
            httpGet:
              path: /ready
              port: 3000
            initialDelaySeconds: 5
            periodSeconds: 10
```

**Paso 8: Service de la API**:

```yaml
# 08-service-api.yaml
apiVersion: v1
kind: Service
metadata:
  name: api-service
  namespace: demo-app
spec:
  type: ClusterIP
  selector:
    app: api
  ports:
    - port: 3000
      targetPort: 3000
      protocol: TCP
```

**Paso 9: Deployment del frontend web**:

```yaml
# 09-deployment-web.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
  namespace: demo-app
  labels:
    app: web
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    metadata:
      labels:
        app: web
        version: v1
    spec:
      containers:
        - name: nginx
          image: nginx:1.25-alpine
          ports:
            - containerPort: 80
          resources:
            requests:
              cpu: "100m"
              memory: "128Mi"
            limits:
              cpu: "250m"
              memory: "256Mi"
          livenessProbe:
            httpGet:
              path: /
              port: 80
            initialDelaySeconds: 10
            periodSeconds: 15
          readinessProbe:
            httpGet:
              path: /
              port: 80
            initialDelaySeconds: 5
            periodSeconds: 5
```

**Paso 10: Service del frontend web**:

```yaml
# 10-service-web.yaml
apiVersion: v1
kind: Service
metadata:
  name: web-service
  namespace: demo-app
spec:
  type: NodePort
  selector:
    app: web
  ports:
    - port: 80
      targetPort: 80
      nodePort: 30080
      protocol: TCP
```

**Paso 11: Ingress (requiere Ingress Controller instalado)**:

```yaml
# 11-ingress.yaml (requiere Ingress Controller instalado)
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-ingress
  namespace: demo-app
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
    - host: demo-app.local
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: web-service
                port:
                  number: 80
          - path: /api
            pathType: Prefix
            backend:
              service:
                name: api-service
                port:
                  number: 3000
```

**Desplegar todo**:

```bash
# Aplicar todos los archivos YAML a la vez
kubectl apply -f 01-namespace.yaml
kubectl apply -f 02-configmap.yaml
kubectl apply -f 03-secret.yaml
kubectl apply -f 04-pvc.yaml
kubectl apply -f 05-deployment-db.yaml
kubectl apply -f 06-service-db.yaml
kubectl apply -f 07-deployment-api.yaml
kubectl apply -f 08-service-api.yaml
kubectl apply -f 09-deployment-web.yaml
kubectl apply -f 10-service-web.yaml
kubectl apply -f 11-ingress.yaml

# O más simple: aplicar todo el directorio
kubectl apply -f .

# Verificar el estado
kubectl get all -n demo-app

# Ver los Pods con sus estados
watch kubectl get pods -n demo-app

# Ver logs de la API
kubectl logs -n demo-app -l app=api --tail=20 -f

# Probar desde dentro del cluster con un pod temporal
kubectl run -it --rm debug --image=curlimages/curl --restart=Never -n demo-app -- \
  curl http://api-service:3000/health

# Acceder al frontend (NodePort en minikube)
minikube service web-service -n demo-app --url
```

---

### 8.7.6 Kubernetes Avanzado: Helm, RBAC, HPA y Network Policies

#### Helm: El Gestor de Paquetes de Kubernetes

Helm es a Kubernetes lo que `apt` es a Debian o `npm` a Node.js. Los **Charts** de Helm son paquetes que contienen todos los manifiestos YAML necesarios para desplegar una aplicación completa, con valores parametrizables.

```bash
# Instalar Helm
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# Añadir repositorio de charts
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update

# Instalar PostgreSQL con valores personalizados
helm install my-postgres bitnami/postgresql \
  --set auth.username=tasksflow \
  --set auth.password=SuperSecret \
  --set auth.database=tasksflow \
  --set primary.persistence.size=10Gi

# Listar releases instalados
helm list

# Ver valores por defecto de un chart
helm show values bitnami/postgresql

# Actualizar release
helm upgrade my-postgres bitnami/postgresql \
  --set primary.resources.limits.memory=512Mi

# Rollback
helm rollback my-postgres 1

# Desinstalar
helm uninstall my-postgres
```

**Crear tu propio Chart para TasksFlow:**

```bash
helm create tasksflow
# Crea la estructura:
# tasksflow/
# ├── Chart.yaml          # Metadatos del chart
# ├── values.yaml         # Valores por defecto
# ├── templates/          # Manifiestos YAML con templates Go
# │   ├── deployment.yaml
# │   ├── service.yaml
# │   ├── configmap.yaml
# │   └── _helpers.tpl    # Funciones auxiliares
# └── .helmignore
```

Modifica `tasksflow/values.yaml`:

```yaml
replicaCount: 3

image:
  repository: ghcr.io/usuario/tasksflow-api
  tag: latest
  pullPolicy: Always

service:
  type: ClusterIP
  port: 3000

resources:
  limits:
    cpu: 500m
    memory: 256Mi
  requests:
    cpu: 100m
    memory: 64Mi

postgresql:
  enabled: true
  auth:
    username: tasksflow
    password: SuperSecret
    database: tasksflow
  primary:
    persistence:
      size: 5Gi

redis:
  enabled: true
  architecture: standalone
  auth:
    enabled: false
```

Modifica `tasksflow/templates/deployment.yaml` para usar las variables del `values.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "tasksflow.fullname" . }}
  labels:
    {{- include "tasksflow.labels" . | nindent 4 }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      {{- include "tasksflow.selectorLabels" . | nindent 6 }}
  template:
    spec:
      containers:
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          ports:
            - containerPort: {{ .Values.service.port }}
          resources:
            {{- toYaml .Values.resources | nindent 12 }}
```

Desplegar tu chart:

```bash
helm install tasksflow ./tasksflow -n tasksflow --create-namespace
```

---

#### RBAC: Control de Acceso Basado en Roles

Kubernetes RBAC controla **quién** (Subject: usuario, grupo, ServiceAccount) puede hacer **qué** (Verb: get, list, create, delete, watch) sobre **qué** (Resource: pods, deployments, secrets) en **qué namespace**.

```yaml
# Role: permisos para el namespace tasksflow
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: tasksflow
  name: api-role
rules:
  - apiGroups: [""]
    resources: ["pods", "services", "configmaps"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["secrets"]
    verbs: ["get"]
    resourceNames: ["db-secret"]

---
# RoleBinding: asignar el Role a la ServiceAccount de la API
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  namespace: tasksflow
  name: api-rolebinding
subjects:
  - kind: ServiceAccount
    name: api-sa
    namespace: tasksflow
roleRef:
  kind: Role
  name: api-role
  apiGroup: rbac.authorization.k8s.io

---
# ServiceAccount para la API (no usar la default)
apiVersion: v1
kind: ServiceAccount
metadata:
  namespace: tasksflow
  name: api-sa
```

Asocia la ServiceAccount al Deployment:

```yaml
spec:
  template:
    spec:
      serviceAccountName: api-sa
      containers:
        - name: api
          image: tasksflow-api:latest
```

---

#### HPA: Horizontal Pod Autoscaler

El HPA escala automáticamente el número de réplicas de un Deployment basándose en métricas de CPU, memoria o métricas personalizadas.

```bash
# Requisito: Metrics Server instalado en el cluster
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

# Crear HPA para la API
kubectl autoscale deployment api -n tasksflow \
  --cpu-percent=70 \
  --min=2 \
  --max=10

# O mediante YAML:
```

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: api-hpa
  namespace: tasksflow
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: api
  minReplicas: 2
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 80
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300  # Esperar 5 min antes de reducir
      policies:
        - type: Pods
          value: 1
          periodSeconds: 60
    scaleUp:
      stabilizationWindowSeconds: 0    # Escalar inmediatamente si hace falta
      policies:
        - type: Pods
          value: 2
          periodSeconds: 30
```

```bash
# Ver estado del HPA
kubectl get hpa -n tasksflow

# NAME      REFERENCE        TARGETS         MINPODS   MAXPODS   REPLICAS
# api-hpa   Deployment/api   cpu: 45%/70%    2         10        3

# Generar carga para probar
kubectl run -it --rm load-generator --image=busybox -n tasksflow -- \
  /bin/sh -c "while true; do wget -q -O- http://api-svc:3000/health; done"
```

---

#### Network Policies: Firewall entre Pods

Por defecto, cualquier Pod puede comunicarse con cualquier otro Pod en el cluster. Las Network Policies definen reglas de firewall granulares.

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: api-network-policy
  namespace: tasksflow
spec:
  podSelector:
    matchLabels:
      app: api
  policyTypes:
    - Ingress
    - Egress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: web        # Solo el frontend puede llamar a la API
      ports:
        - port: 3000
          protocol: TCP
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: monitoring  # Prometheus desde namespace monitoring
      ports:
        - port: 3000
          protocol: TCP
  egress:
    - to:
        - podSelector:
            matchLabels:
              app: postgres    # La API solo necesita hablar con postgres y redis
      ports:
        - port: 5432
          protocol: TCP
    - to:
        - podSelector:
            matchLabels:
              app: redis
      ports:
        - port: 6379
          protocol: TCP
    - to:                     # Permitir DNS
        - namespaceSelector: {}
          podSelector:
            matchLabels:
              k8s-app: kube-dns
      ports:
        - port: 53
          protocol: UDP
```

```bash
# Aplicar
kubectl apply -f network-policy.yaml

# Verificar que la política se aplica
kubectl get networkpolicy -n tasksflow
```

---

## 8.8 Comparativa Swarm vs Kubernetes

### 8.8.1 Tabla comparativa extensa

| Característica | Docker Swarm | Kubernetes |
|---------------|--------------|------------|
| **Origen** | Docker Inc. (2016) | Google (2014), CNCF |
| **Instalación** | Incluido en Docker Engine (0 pasos extra) | Múltiples componentes (kubeadm, binarios, o distro) |
| **Complejidad** | Baja. Se aprende en horas. | Alta. Se requieren semanas/meses. |
| **Configuración** | Mínima. Funciona out-of-the-box. | Extensa. Muchas opciones, mucho YAML. |
| **Lenguaje de definición** | Docker Compose + extensiones `deploy:` | YAML nativo de Kubernetes (Deployment, Service, etc.) |
| **Curva de aprendizaje** | Suave. Familiar si conoces Docker. | Empinada. Muchos conceptos nuevos. |
| **Nodos manager** | Raft consensus. 1-7 recomendado. | Control plane con etcd. 3-5 recomendado. |
| **Networking** | Overlay nativo (VXLAN). Routing mesh integrado. | CNI plugins (Calico, Flannel, Cilium, etc.) — elegible. |
| **Service Discovery** | DNS interno automático. | DNS interno + Services (ClusterIP). |
| **Balanceo de carga** | Routing mesh (IPVS integrado). | Services + Ingress + balanceadores externos. |
| **Secrets** | Encriptados en Raft log (AES-256-GCM). | Codificados en base64 (requiere etcd encryption adicional). |
| **Configs** | Sí (no encriptados). | ConfigMaps. |
| **Rolling updates** | Sí, configurable (delay, parallelism, failure action). | Sí, Deployment nativo + estrategias avanzadas. |
| **Rollback** | `docker service rollback` (limitado en historial). | `kubectl rollout undo` (historial completo). |
| **Auto-scaling** | No nativo. Requiere herramientas externas. | HPA (Horizontal Pod Autoscaler) nativo. |
| **Volúmenes** | Volume drivers (local, NFS, cloud). | PV + PVC + StorageClasses (provisionamiento dinámico). |
| **Health checks** | HEALTHCHECK en Dockerfile. | livenessProbe, readinessProbe, startupProbe. |
| **Namespaces** | No (stacks como alternativa ligera). | Sí (aislamiento lógico completo). |
| **RBAC** | No nativo en Swarm standalone. Sí en Docker EE/UCP. | Sí, nativo y granular (Roles, ClusterRoles, Bindings). |
| **Dashboards** | Portainer, Swarmpit (externos). | Kubernetes Dashboard (oficial), Lens, Octant, etc. |
| **Ecosistema** | Limitado. | Inmenso. Helm, Prometheus, Istio, ArgoCD, cert-manager, etc. |
| **Comunidad** | Pequeña. En declive. | ENORME. La más grande después de Linux. |
| **Cloud providers** | Escaso soporte. | Soporte nativo en AWS (EKS), GCP (GKE), Azure (AKS). |
| **CI/CD integración** | Básica. Docker Hub + webhooks. | GitHub Actions, GitLab CI, ArgoCD, Flux, Jenkins X. |
| **Service Mesh** | No nativo. | Istio, Linkerd, Consul Connect. |
| **Serverless** | No. | Knative, OpenFaaS, Kubeless. |
| **Escalabilidad** | Cientos de nodos. | Miles de nodos. |
| **Multi-tenancy** | Limitado. | Namespaces + NetworkPolicies + ResourceQuotas + RBAC. |
| **Custom Resources** | No. | CRDs (Custom Resource Definitions) + Operators. |
| **Admisión dinámica** | No. | Admission Controllers + Webhooks. |
| **Logging** | `docker service logs` (básico). | EFK/Loki stack. |
| **Monitoreo** | Externo (Prometheus + Grafana). | Prometheus Operator + Grafana (estándares). |
| **Actualizaciones del cluster** | Complejo. Downtime de managers. | `kubeadm upgrade` (gestionado). |
| **Documentación** | Buena pero limitada. | Excelente y exhaustiva. |
| **Madurez** | Madura pero estancada. | Hiper-madura, rápida evolución. |
| **Licencia** | Apache 2.0 (open source). | Apache 2.0 (open source). |
| **Soporte comercial** | Docker Business (antes Mirantis). | Todas las grandes clouds + Red Hat (OpenShift) + VMware (Tanzu). |
| **Casos de uso** | Equipos pequeños, on-premise, simple HA. | Empresas, multi-cloud, escala masiva, microservicios complejos. |

### 8.8.2 Cuándo usar Docker Swarm

Swarm brilla en escenarios concretos:

1. **Equipos pequeños (1-5 personas)** que ya conocen Docker y necesitan HA pero no tienen tiempo de aprender Kubernetes.
2. **Proyectos on-premise** con 2-5 nodos donde la simplicidad es prioridad absoluta.
3. **Aplicaciones monolíticas o pocos microservicios** (5-15 servicios) sin necesidades complejas de networking.
4. **Edge computing y IoT**: por su bajo consumo de recursos y facilidad de gestión.
5. **Desarrollo local y CI/CD ligero**: levantar un cluster para tests de integración es trivial.

```bash
# Crear un cluster Swarm de 3 nodos: ~5 minutos
# (contando el tiempo de arranque de las VMs)

# En el manager:
docker swarm init --advertise-addr 192.168.1.10

# En cada worker:
docker swarm join --token <token> 192.168.1.10:2377

# Desplegar aplicación:
docker stack deploy -c docker-compose.yml myapp

# ¡Listo! Cluster HA con balanceo de carga, secrets,
# rolling updates, y routing mesh en 5 minutos.
```

### 8.8.3 Cuándo usar Kubernetes

Kubernetes es la elección correcta cuando:

1. **Escalas masivamente**: decenas o cientos de nodos, miles de Pods. K8s fue diseñado para esto.
2. **Multi-cloud o hybrid-cloud**: EKS, GKE, AKS ofrecen Kubernetes gestionado. Portabilidad real entre clouds.
3. **Ecosistema rico**: necesitas Helm charts, operators, service mesh (Istio), GitOps (ArgoCD), certificados automáticos (cert-manager), tracing (Jaeger), etc.
4. **Equipos grandes (10+ personas)**: diferentes equipos pueden trabajar en diferentes namespaces con RBAC y NetworkPolicies.
5. **Despliegues avanzados**: canary, blue-green, A/B testing con control de tráfico fino (Istio, Flagger).
6. **Auto-scaling**: escalado automático basado en CPU, memoria o métricas custom (HPA + KEDA).
7. **Custom Resources y Operators**: necesitas extender Kubernetes con tus propias abstracciones ("base de datos como servicio", "aplicación como recurso").

```bash
# Desplegar el mismo stack en K8s (manifiestos YAML completos)
# ~520 líneas de YAML (vs ~120 en Compose)

kubectl apply -f namespace.yaml
kubectl apply -f configmap.yaml
kubectl apply -f secret.yaml
kubectl apply -f pvc.yaml
kubectl apply -f deployment-db.yaml
kubectl apply -f service-db.yaml
kubectl apply -f deployment-api.yaml
kubectl apply -f service-api.yaml
kubectl apply -f deployment-web.yaml
kubectl apply -f service-web.yaml
kubectl apply -f ingress.yaml

# Configurar autoscaling:
kubectl autoscale deployment api --cpu-percent=70 --min=3 --max=20

# Instalar monitoreo (Prometheus stack):
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm install prometheus prometheus-community/kube-prometheus-stack
```

### 8.8.4 El debate Swarm vs Kubernetes resuelto

```
¿Necesitas más de 50 Pods?                         → Kubernetes
¿Necesitas auto-scaling nativo?                    → Kubernetes
¿Necesitas multi-cloud portability?                → Kubernetes
¿Tu empresa ya usa cloud managed K8s?              → Kubernetes
¿Necesitas GitOps (ArgoCD/Flux)?                   → Kubernetes
¿Necesitas service mesh (Istio)?                   → Kubernetes
¿Tienes un equipo de 2-3 devs con prisa?           → Docker Swarm
¿Es on-premise con 3 servidores?                   → Docker Swarm
¿Quieres levantar HA en 5 minutos?                 → Docker Swarm
¿Ya tienes todo en Docker Compose?                 → Docker Swarm
¿Es un proyecto pequeño que no escalará mucho?     → Docker Swarm
¿No quieres aprender 50 conceptos nuevos?          → Docker Swarm
```

La verdad incómoda: **muchos equipos usan Kubernetes cuando Docker Swarm sería suficiente**. La presión de la industria y el "Kubernetes o nada" ha llevado a una complejidad innecesaria en muchos proyectos. Pero la decisión debe basarse en tus necesidades reales, no en tendencias.

---

## 8.9 Laboratorio

### Objetivo

Desplegar la misma aplicación 3-tier (web + api + db) en Swarm y en Kubernetes. Comparar la experiencia real: comandos, YAML, tiempo de setup, debugging, actualizaciones, secretos.

### Arquitectura de la aplicación

```
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│   FRONTEND   │────►│   BACKEND    │────►│   DATABASE   │
│              │     │              │     │              │
│  Nginx:1.25  │     │  Node.js API │     │ PostgreSQL 16│
│  Puerto 80   │     │  Puerto 3000 │     │  Puerto 5432 │
│              │     │              │     │              │
│  / → index   │     │  /health     │     │  PGUSER=user │
│  /api → api  │     │  /api/users  │     │  PGPASS=sec  │
│              │     │  /api/tasks  │     │              │
└──────────────┘     └──────────────┘     └──────────────┘
    3 réplicas          3 réplicas           1 réplica
```

### Parte A: Despliegue en Docker Swarm

**Paso 1: Preparar el cluster Swarm (3 nodos)**

```bash
# Nodo 1: Manager (192.168.1.10)
docker swarm init --advertise-addr 192.168.1.10
# Guardar el token que se imprime

# Nodo 2: Worker (192.168.1.20)
docker swarm join --token <token> 192.168.1.10:2377

# Nodo 3: Worker (192.168.1.30)
docker swarm join --token <token> 192.168.1.10:2377

# Verificar
docker node ls
```

**Paso 2: Crear secretos**

```bash
echo "SuperSecretPass2026!" | docker secret create db_password -
echo "my-jwt-secret-key-256-bits-long!!" | docker secret create jwt_secret -
echo "debug-mode-off" | docker secret create api_key -
```

**Paso 3: Crear el archivo de stack para Swarm**

```yaml
# lab-swarm/docker-compose.yml
version: '3.9'

services:

  web:
    image: nginx:1.25-alpine
    ports:
      - "80:80"
    networks:
      - frontend
    configs:
      - source: nginx_conf
        target: /etc/nginx/conf.d/default.conf
    environment:
      - API_URL=http://api:3000
    healthcheck:
      test: ["CMD", "wget", "-qO-", "http://localhost/health"]
      interval: 10s
      timeout: 5s
      retries: 3
      start_period: 10s
    deploy:
      replicas: 3
      resources:
        limits:
          cpus: '0.5'
          memory: 256M
        reservations:
          cpus: '0.25'
          memory: 128M
      update_config:
        parallelism: 1
        delay: 10s
        failure_action: rollback
        order: start-first
      restart_policy:
        condition: on-failure
        delay: 5s
        max_attempts: 3

  api:
    image: lab-api:latest
    ports:
      - "3000:3000"
    environment:
      - NODE_ENV=production
      - DB_HOST=db
      - DB_PORT=5432
      - DB_USER=appuser
      - DB_NAME=appdb
    secrets:
      - source: db_password
        target: /run/secrets/db_password
      - source: jwt_secret
        target: /run/secrets/jwt_secret
      - source: api_key
        target: /run/secrets/api_key
    networks:
      - frontend
      - backend
    healthcheck:
      test: ["CMD", "wget", "-qO-", "http://localhost:3000/health"]
      interval: 15s
      timeout: 5s
      retries: 3
      start_period: 20s
    deploy:
      replicas: 3
      resources:
        limits:
          cpus: '1'
          memory: 512M
        reservations:
          cpus: '0.5'
          memory: 256M
      update_config:
        parallelism: 1
        delay: 15s
        failure_action: rollback
        order: start-first
      restart_policy:
        condition: any
        delay: 5s

  db:
    image: postgres:16-alpine
    environment:
      - POSTGRES_USER=appuser
      - POSTGRES_DB=appdb
      - POSTGRES_PASSWORD_FILE=/run/secrets/db_password
    secrets:
      - source: db_password
        target: /run/secrets/db_password
    volumes:
      - db_data:/var/lib/postgresql/data
    networks:
      - backend
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U appuser -d appdb"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 15s
    deploy:
      replicas: 1
      placement:
        constraints:
          - node.labels.tier == database
      resources:
        limits:
          cpus: '1'
          memory: 1024M
        reservations:
          cpus: '0.5'
          memory: 512M
      update_config:
        parallelism: 1
        delay: 30s
        order: stop-first
      restart_policy:
        condition: any
        delay: 10s

networks:
  frontend:
    driver: overlay
    attachable: true
  backend:
    driver: overlay
    internal: true

volumes:
  db_data:
    driver: local

secrets:
  db_password:
    external: true
  jwt_secret:
    external: true
  api_key:
    external: true

configs:
  nginx_conf:
    file: ./nginx.conf
```

**Configuración de Nginx para el laboratorio**:

```nginx
# lab-swarm/nginx.conf
server {
    listen 80;
    server_name _;

    location / {
        root /usr/share/nginx/html;
        index index.html;
        try_files $uri $uri/ =404;
    }

    location /api/ {
        proxy_pass http://api:3000/;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_cache_bypass $http_upgrade;
    }

    location /health {
        return 200 '{"status":"ok","service":"web"}';
        add_header Content-Type application/json;
    }
}
```

**Paso 4: Desplegar el stack**

```bash
# Etiquetar un nodo para la base de datos
docker node update --label-add tier=database worker-02

# Desplegar el stack
docker stack deploy -c docker-compose.yml lab

# Verificar
docker stack services lab
docker service ps lab_web
docker service ps lab_api
docker service ps lab_db

# Ver logs
docker service logs lab_api
docker service logs lab_db

# Probar el routing mesh
curl http://192.168.1.10:80/health
curl http://192.168.1.20:80/health
curl http://192.168.1.30:80/health
# Debería funcionar en cualquier IP, no solo donde corre web

# Probar la API a través del proxy de nginx
curl http://192.168.1.10/api/health
```

**Paso 5: Rolling update en Swarm**

```bash
# Actualizar la imagen de la API
docker service update --image lab-api:v2 lab_api

# Monitorear el progreso
watch docker service ps lab_api

# Si algo sale mal, hacer rollback
docker service rollback lab_api
```

**Paso 6: Simular fallo de un worker**

```bash
# Ver dónde están corriendo las réplicas de web
docker service ps lab_web

# Poner un worker en drain (simula caída)
docker node update --availability drain worker-01

# Ver cómo Swarm reprograma las réplicas
docker service ps lab_web
# Las réplicas que estaban en worker-01 ahora aparecen en worker-02 o manager-01

# Reactivar el worker
docker node update --availability active worker-01
```

### Parte B: Despliegue en Kubernetes

**Paso 1: Crear el namespace y secretos**

```bash
kubectl create namespace lab-app

# Crear secretos (equivale a docker secret create)
kubectl create secret generic db-credentials \
  --from-literal=username=appuser \
  --from-literal=password=SuperSecretPass2026! \
  -n lab-app

kubectl create secret generic jwt-secret \
  --from-literal=key=my-jwt-secret-key-256-bits-long!! \
  -n lab-app

kubectl create secret generic api-key \
  --from-literal=key=debug-mode-off \
  -n lab-app
```

**Paso 2: Crear ConfigMap para la aplicación**

```bash
kubectl create configmap app-config \
  --from-literal=APP_ENV=production \
  --from-literal=DB_HOST=db-service \
  --from-literal=DB_PORT=5432 \
  --from-literal=DB_NAME=appdb \
  -n lab-app

# Crear ConfigMap para nginx.conf desde archivo
kubectl create configmap nginx-config \
  --from-file=nginx.conf \
  -n lab-app
```

**Paso 3: Crear PVC para la base de datos**

```yaml
# lab-k8s/01-pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: db-pvc
  namespace: lab-app
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
  storageClassName: standard
```

**Paso 4: Desplegar la base de datos**

```yaml
# lab-k8s/02-deployment-db.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: db
  namespace: lab-app
  labels:
    app: db
    tier: backend
spec:
  replicas: 1
  selector:
    matchLabels:
      app: db
  template:
    metadata:
      labels:
        app: db
        tier: backend
    spec:
      containers:
        - name: postgres
          image: postgres:16-alpine
          ports:
            - containerPort: 5432
          env:
            - name: POSTGRES_USER
              valueFrom:
                secretKeyRef:
                  name: db-credentials
                  key: username
            - name: POSTGRES_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: db-credentials
                  key: password
            - name: POSTGRES_DB
              valueFrom:
                configMapKeyRef:
                  name: app-config
                  key: DB_NAME
          resources:
            requests:
              cpu: "250m"
              memory: "256Mi"
            limits:
              cpu: "1"
              memory: "1Gi"
          volumeMounts:
            - name: db-storage
              mountPath: /var/lib/postgresql/data
          livenessProbe:
            exec:
              command: ["pg_isready", "-U", "appuser", "-d", "appdb"]
            initialDelaySeconds: 30
            periodSeconds: 10
          readinessProbe:
            exec:
              command: ["pg_isready", "-U", "appuser", "-d", "appdb"]
            initialDelaySeconds: 5
            periodSeconds: 5
      volumes:
        - name: db-storage
          persistentVolumeClaim:
            claimName: db-pvc
---
# lab-k8s/03-service-db.yaml
apiVersion: v1
kind: Service
metadata:
  name: db-service
  namespace: lab-app
spec:
  type: ClusterIP
  selector:
    app: db
  ports:
    - port: 5432
      targetPort: 5432
```

**Paso 5: Desplegar la API**

```yaml
# lab-k8s/04-deployment-api.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api
  namespace: lab-app
  labels:
    app: api
    tier: backend
spec:
  replicas: 3
  selector:
    matchLabels:
      app: api
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    metadata:
      labels:
        app: api
        tier: backend
        version: v1
    spec:
      containers:
        - name: api
          image: lab-api:latest
          ports:
            - containerPort: 3000
          env:
            - name: NODE_ENV
              valueFrom:
                configMapKeyRef:
                  name: app-config
                  key: APP_ENV
            - name: DB_HOST
              valueFrom:
                configMapKeyRef:
                  name: app-config
                  key: DB_HOST
            - name: DB_PORT
              valueFrom:
                configMapKeyRef:
                  name: app-config
                  key: DB_PORT
            - name: DB_NAME
              valueFrom:
                configMapKeyRef:
                  name: app-config
                  key: DB_NAME
            - name: DB_USER
              valueFrom:
                secretKeyRef:
                  name: db-credentials
                  key: username
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: db-credentials
                  key: password
            - name: JWT_SECRET
              valueFrom:
                secretKeyRef:
                  name: jwt-secret
                  key: key
          resources:
            requests:
              cpu: "250m"
              memory: "256Mi"
            limits:
              cpu: "1"
              memory: "512Mi"
          livenessProbe:
            httpGet:
              path: /health
              port: 3000
            initialDelaySeconds: 15
            periodSeconds: 20
          readinessProbe:
            httpGet:
              path: /ready
              port: 3000
            initialDelaySeconds: 5
            periodSeconds: 10
---
# lab-k8s/05-service-api.yaml
apiVersion: v1
kind: Service
metadata:
  name: api-service
  namespace: lab-app
spec:
  type: ClusterIP
  selector:
    app: api
  ports:
    - port: 3000
      targetPort: 3000
```

**Paso 6: Desplegar el frontend web**

```yaml
# lab-k8s/06-deployment-web.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
  namespace: lab-app
  labels:
    app: web
    tier: frontend
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    metadata:
      labels:
        app: web
        tier: frontend
        version: v1
    spec:
      containers:
        - name: nginx
          image: nginx:1.25-alpine
          ports:
            - containerPort: 80
          volumeMounts:
            - name: nginx-config
              mountPath: /etc/nginx/conf.d/default.conf
              subPath: nginx.conf
          resources:
            requests:
              cpu: "100m"
              memory: "128Mi"
            limits:
              cpu: "500m"
              memory: "256Mi"
          livenessProbe:
            httpGet:
              path: /health
              port: 80
            initialDelaySeconds: 10
            periodSeconds: 15
          readinessProbe:
            httpGet:
              path: /health
              port: 80
            initialDelaySeconds: 5
            periodSeconds: 5
      volumes:
        - name: nginx-config
          configMap:
            name: nginx-config
---
# lab-k8s/07-service-web.yaml
apiVersion: v1
kind: Service
metadata:
  name: web-service
  namespace: lab-app
spec:
  type: NodePort
  selector:
    app: web
  ports:
    - port: 80
      targetPort: 80
      nodePort: 30080
```

**Paso 7: Desplegar todo en Kubernetes**

```bash
# Aplicar todos los manifiestos
kubectl apply -f lab-k8s/

# Verificar despliegue
kubectl get all -n lab-app
kubectl get pods -n lab-app -o wide

# Ver logs
kubectl logs -n lab-app deployment/api --tail=20 -f
kubectl logs -n lab-app deployment/db

# Probar la API desde dentro del cluster
kubectl run -it --rm debug --image=curlimages/curl --restart=Never -n lab-app -- \
  curl http://api-service:3000/health

# Probar el frontend
kubectl run -it --rm debug --image=curlimages/curl --restart=Never -n lab-app -- \
  curl http://web-service/health

# Acceder desde fuera (NodePort)
# Si usas Minikube:
minikube service web-service -n lab-app --url

# Si usas Kind con port-forward:
kubectl port-forward -n lab-app svc/web-service 8080:80
# Luego: curl http://localhost:8080/health
```

**Paso 8: Rolling update y rollback en Kubernetes**

```bash
# Actualizar la imagen de la API
kubectl set image deployment/api api=lab-api:v2 -n lab-app

# Monitorear el progreso
kubectl rollout status deployment/api -n lab-app

# Ver historial
kubectl rollout history deployment/api -n lab-app

# Si algo sale mal, hacer rollback
kubectl rollout undo deployment/api -n lab-app

# Rollback a una revisión específica
kubectl rollout undo deployment/api --to-revision=1 -n lab-app

# Forzar reinicio de todos los Pods (útil para debugging)
kubectl rollout restart deployment/web -n lab-app
kubectl rollout restart deployment/api -n lab-app
```

**Paso 9: Escalar en Kubernetes**

```bash
# Escalar la API manualmente
kubectl scale deployment api --replicas=5 -n lab-app

# Configurar autoscaling horizontal (requiere metrics-server)
kubectl autoscale deployment api \
  --cpu-percent=70 \
  --min=2 \
  --max=10 \
  -n lab-app

# Ver el HPA
kubectl get hpa -n lab-app

# Probar autoescalado
kubectl run -it --rm load-generator --image=busybox:1.36 --restart=Never -n lab-app -- \
  /bin/sh -c "while true; do wget -q -O- http://api-service:3000/health; done"
```

### Parte C: Comparativa de la experiencia

**Tabla de comparación del laboratorio**:

| Aspecto | Docker Swarm | Kubernetes |
|--------|-------------|------------|
| **Tiempo de setup del cluster** | ~2 minutos | ~5-10 minutos (Minikube/Kind) |
| **Líneas de YAML para el stack** | ~120 líneas (1 archivo) | ~400 líneas (7 archivos) |
| **Creación de secretos** | `echo "x" \| docker secret create name -` | `kubectl create secret generic` |
| **Despliegue** | `docker stack deploy -c file.yml name` | `kubectl apply -f ./dir/` |
| **Ver estado** | `docker stack ps name` | `kubectl get pods -n ns` |
| **Logs** | `docker service logs name_svc` | `kubectl logs -n ns deployment/name` |
| **Actualizar imagen** | `docker service update --image x:v2 svc` | `kubectl set image deployment/n c=v2` |
| **Rollback** | `docker service rollback svc` | `kubectl rollout undo deployment/n` |
| **Escalar** | `docker service scale svc=5` | `kubectl scale deployment n --replicas=5` |
| **Auto-scaling** | No disponible nativamente | `kubectl autoscale deployment...` |
| **Health checks** | HEALTHCHECK en Dockerfile | livenessProbe + readinessProbe |
| **Balanceo de carga** | Routing mesh automático | Service + Ingress Controller |
| **Complejidad percibida** | Baja | Alta |
| **Flexibilidad** | Media | Muy alta |
| **Curva de aprendizaje** | 1-2 días | 2-4 semanas |

### Conclusiones del laboratorio

Después de completar este laboratorio, deberías tener una comprensión práctica de ambas plataformas:

1. **Swarm** es significativamente más simple de configurar y operar. En menos de 10 minutos puedes tener un cluster HA corriendo una aplicación 3-tier con secretos. La contrapartida es menor flexibilidad: no tienes auto-scaling nativo, el historial de rollbacks es limitado, y el ecosistema de herramientas es más reducido.

2. **Kubernetes** requiere más inversión inicial en aprendizaje y configuración (más archivos YAML, más conceptos). Pero a cambio obtienes un ecosistema riquísimo: auto-scaling, operadores, GitOps, service mesh, monitoreo integrado, y la tranquilidad de que prácticamente cualquier problema que enfrentes ya fue resuelto por la comunidad.

3. **La decisión no es binaria**. Muchas empresas usan Swarm en entornos pequeños/edge y Kubernetes en producción principal. Algunas incluso migran gradualmente de Swarm a K8s cuando la complejidad lo justifica.

4. **Lo más importante**: ambos resuelven el mismo problema fundamental — orquestar contenedores a través de múltiples hosts. Ambos funcionan. Ambos están probados en producción. La elección depende de tu contexto específico: tamaño del equipo, complejidad de la aplicación, requisitos de escalabilidad, y presupuesto de tiempo para aprender.

---

## Resumen del capítulo

En este capítulo hemos recorrido un largo camino:

- **Entendimos por qué** la orquestación es necesaria: límites de recursos, falta de HA, sin balanceo de carga integrado, despliegues manuales, secretos inseguros.
- **Dominamos Docker Swarm**: arquitectura (managers, workers, Raft), servicios (create, update, scale, rollback), stacks (Compose en Swarm), secrets y configs, routing mesh.
- **Exploramos Kubernetes**: Control Plane, worker nodes, flujo de peticiones, Pods, Deployments, Services, Ingress, ConfigMaps, Secrets, PV/PVC, Namespaces.
- **Comparamos** ambas plataformas en una tabla exhaustiva con 30+ criterios.
- **Desplegamos** la misma aplicación 3-tier en ambas plataformas, comparando cada paso.

Ahora tienes las herramientas para tomar decisiones informadas sobre orquestación. Recuerda: no hay una respuesta correcta universal. Hay la respuesta correcta para TU contexto.

---

> *"La orquestación no se trata de la herramienta. Se trata de mantener tus aplicaciones vivas cuando el mundo conspira para tumbarlas."*

---

*Fin del Capítulo 8*

---

← [Capítulo anterior](capitulo-07-compose.md) | [Inicio](README.md) | [Capítulo siguiente →](capitulo-09-cicd.md)
