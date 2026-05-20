# Capítulo 6: Redes en Docker

> *"La red es la computadora."* — John Gage, Sun Microsystems

---

En los capítulos anteriores aprendiste a crear, ejecutar y gestionar contenedores. Pero un contenedor aislado es como un castillo sin puentes: poderoso, pero desconectado del mundo. Docker transforma contenedores individuales en sistemas distribuidos precisamente a través de sus capacidades de red.

Este capítulo es probablemente el más importante del libro. Las redes en Docker no son un apéndice ni un tema avanzado: son el tejido conectivo que convierte un conjunto de contenedores en una arquitectura de microservicios, una aplicación distribuida o un clúster de Swarm. Si solo entiendes redes en Docker a medias, construirás castillos en el aire.

Vamos a recorrer cada driver de red, cada concepto, cada comando y cada escenario práctico. Desde el bridge por defecto hasta overlay en Swarm, desde el troubleshooting hasta el laboratorio final. No dejes nada sin entender.

---

## 6.1 El Modelo de Red de Docker: Container Network Model (CNM)

### 6.1.1 ¿Qué es el CNM?

El **Container Network Model (CNM)** es la especificación que define cómo Docker gestiona las redes. No es código, es un modelo conceptual que establece la arquitectura de networking. Fue diseñado por Docker en 2015 para estandarizar y hacer extensible el subsistema de red.

El CNM define tres componentes fundamentales. Piensa en ellos como el vocabulario mínimo para describir cualquier topología de red en Docker:

```
+------------------------------------------------------------------+
|                     CONTAINER NETWORK MODEL                       |
|                                                                   |
|    +-------------+        +-------------+        +-------------+  |
|    |             |        |             |        |             |  |
|    |   SANDBOX   |<------>|  ENDPOINT   |<------>|   NETWORK   |  |
|    |             |        |             |        |             |  |
|    +-------------+        +-------------+        +-------------+  |
|                                                                   |
|   Namespace de red       Conexión entre          Grupo de         |
|   aislado del           sandbox y red.         endpoints que      |
|   contenedor.           Par (veth).            se pueden          |
|                                                 comunicar.        |
+------------------------------------------------------------------+
```

### 6.1.2 Los tres componentes en detalle

#### Sandbox

Un **Sandbox** es un contenedor de nombres de red (network namespace) aislado. En el kernel de Linux, un network namespace es una copia independiente de toda la pila de red: interfaces, tablas de enrutamiento, reglas de iptables, sockets. Lo que ocurre dentro de un namespace no afecta a otro.

Cuando ejecutas `docker run`, Docker crea un sandbox para ese contenedor. Es lo que ves cuando ejecutas:

```bash
docker exec mi_contenedor ip addr
```

Esa salida (`lo`, `eth0@ifXX`) pertenece al sandbox del contenedor. Fuera de ese sandbox, en el namespace raíz del host, esas interfaces no existen con esos nombres. Esta es la magia del aislamiento de red.

Cada contenedor tiene **exactamente un sandbox**. Un sandbox puede tener múltiples endpoints (si el contenedor está conectado a varias redes).

#### Endpoint

Un **Endpoint** es la conexión entre un sandbox y una red. Físicamente, en Linux, un endpoint se implementa como un par de interfaces virtuales Ethernet (veth pair). Una interfaz del par vive en el sandbox del contenedor (normalmente llamada `eth0`), y la otra vive en el namespace de red donde reside el bridge o el switch virtual correspondiente.

```
+-------------------+                +------------------------+
|     SANDBOX       |                |       NETWORK          |
|   (namespace)     |                |    (bridge/overlay)    |
|                   |                |                        |
|   eth0 -----------|-- veth pair ---|-------- vethXXX        |
|   192.168.1.2     |                |                        |
|                   |                |                        |
+-------------------+                +------------------------+
```

Cada endpoint pertenece exactamente a una red y exactamente a un sandbox. Un contenedor con 3 redes tiene 3 endpoints (y por tanto 3 pares veth).

#### Network

Una **Network** es un grupo de endpoints que pueden comunicarse entre sí directamente. Es un dominio de broadcast L2 aislado. Las implementaciones concretas de una Network son los drivers de red: bridge, overlay, macvlan, etc.

Desde el punto de vista del CNM, una red es un objeto lógico que agrupa endpoints. Docker le asigna un ID, un nombre, un driver, una subred IP y opciones de configuración.

### 6.1.3 libnetwork: la implementación del CNM

**libnetwork** es la biblioteca escrita en Go que implementa el CNM en Docker. Está integrada en el daemon de Docker (`dockerd`) y expone una API para que los drivers de red se registren y gestionen los recursos.

```
+----------------------------------------------------------+
|                       dockerd                            |
|                                                          |
|   +------------------+     +---------------------------+ |
|   |   Docker Engine  |---->|       libnetwork          | |
|   |   (CLI -> API)   |     |                           | |
|   +------------------+     |  +---------------------+  | |
|                            |  | Controller (manager)|  | |
|                            |  +----------+----------+  | |
|                            |             |              | |
|                            |    +--------+--------+     | |
|                            |    |                 |     | |
|                            |  +--+---+  +------+--+--+  | |
|                            |  |Bridge|  |Overlay   |  |  | |
|                            |  |Driver|  |Driver    |  |  | |
|                            |  +------+  +----------+  |  | |
|                            |  |Macvlan| |Host| |None| |  | |
|                            |  +-------+ +----+ +----+ |  | |
|                            +---------------------------+  | |
+----------------------------------------------------------+
```

libnetwork provee:
- Un **Controller** que gestiona el ciclo de vida de redes, endpoints y sandboxes.
- Una abstracción de **Driver** que permite implementar diferentes tipos de red.
- La gestión de **IPAM** (IP Address Management): asignación de subredes y direcciones IP.

Los drivers nativos incluidos en Docker son: `bridge`, `host`, `overlay`, `macvlan`, `ipvlan`, `none`. Existen también drivers de terceros (Weave, Calico, Flannel, Cilium) que se registran como plugins.

### 6.1.4 El flujo de vida de una red

Cuando creas una red con `docker network create`, libnetwork:

1. Registra la red en su almacenamiento interno (boltdb en `/var/lib/docker/network/files/`).
2. Invoca al driver correspondiente para que cree la infraestructura (por ejemplo, el driver bridge crea un bridge Linux).
3. Configura IPAM para la subred especificada.
4. Reserva el gateway y el pool de direcciones.

Cuando conectas un contenedor a esa red (`docker run --network`), libnetwork:

1. Crea un sandbox (network namespace) para el contenedor si no existe.
2. Crea un endpoint en la red.
3. Conecta el endpoint al sandbox mediante un par veth.
4. Asigna una IP al endpoint desde el pool de IPAM.
5. Configura rutas, DNS y reglas de iptables.

### 6.1.5 VETH Pairs: el pegamento de las redes Docker

Cuando un contenedor se conecta a una red, libnetwork crea un par de interfaces virtuales Ethernet (veth pair). Una veth pair es como un cable virtual con dos extremos: lo que entra por un extremo sale por el otro, y viceversa.

```
+---------------------------+       +---------------------------+
|    NAMESPACE CONTENEDOR   |       |    NAMESPACE DEL HOST     |
|                           |       |                           |
|    eth0 (veth0) <=========|=======|========> veth1XXXX        |
|    172.18.0.2/16          |       |    (conectada al bridge)  |
|                           |       |                           |
+---------------------------+       +---------------------------+
           EXTREMO "PEER"                   EXTREMO "HOST"
      (dentro del contenedor)        (en el namespace raíz)
```

Para ver los pares veth desde el host:

```bash
$ ip link show | grep veth
8: vethabc123@if7: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 ...
    link/ether aa:bb:cc:dd:ee:ff brd ff:ff:ff:ff:ff:ff link-netnsid 0
```

El `@if7` en `vethabc123@if7` indica el índice de la interfaz "hermana" en el otro namespace. El kernel mantiene esta relación: son dos caras de la misma moneda.

Puedes mapear un veth a su contenedor:

```bash
$ docker inspect --format '{{.State.Pid}}' contenedor
12345
$ sudo nsenter -t 12345 -n ip link show type veth
```

### 6.1.6 Arquitectura interna de libnetwork

libnetwork se estructura en tres capas que corresponden a los tres componentes del CNM:

```
+-------------------------------------------------------+
|                   libnetwork                          |
|                                                       |
|  +-----------------------------------------------+   |
|  |            Controller (NetworkController)      |   |
|  |                                               |   |
|  |  - Registra y gestiona drivers de red         |   |
|  |  - Gestiona el almacenamiento (boltdb)        |   |
|  |  - Orquesta la creación de redes, endpoints   |   |
|  |  - Implementa IPAM (gestión de direcciones)   |   |
|  +-----------------------+-----------------------+   |
|                          |                           |
|         +----------------+----------------+         |
|         |                |                |         |
|  +------+------+  +-----+------+  +-----+------+   |
|  |   Network   |  |   Network  |  |   Network  |   |
|  |   (objeto)  |  |   (objeto) |  |   (objeto) |   |
|  +------+------+  +-----+------+  +-----+------+   |
|         |                |                |         |
|  +------+------+  +-----+------+  +-----+------+   |
|  |  Endpoints  |  |  Endpoints |  |  Endpoints |   |
|  +------+------+  +-----+------+  +-----+------+   |
|         |                |                |         |
|  +------+------+  +-----+------+  +-----+------+   |
|  |  Sandboxes  |  |  Sandboxes |  |  Sandboxes |   |
|  +-------------+  +------------+  +------------+   |
+-------------------------------------------------------+
```

El **Controller** es el singleton que gestiona todo el subsistema de red. Se inicializa cuando arranca `dockerd` y mantiene referencias a todos los drivers registrados.

Los **Network objects** mantienen la configuración (subred, gateway, opciones) y delegan en el driver para la implementación concreta.

Cada **Sandbox** tiene un `OS Sbox` subyacente que gestiona las llamadas al kernel para crear/destruir network namespaces.

### 6.1.7 El almacén de datos de libnetwork

Docker persiste la configuración de redes en archivos JSON dentro de `/var/lib/docker/network/files/`:

```bash
$ sudo ls /var/lib/docker/network/files/
local-kv.db
```

El archivo `local-kv.db` es una base de datos BoltDB (un motor key-value embebido en Go, similar a SQLite pero para pares clave-valor). Contiene toda la información de las redes: configuración, endpoints, IPs asignadas.

```bash
# Inspeccionar el contenido (requiere boltdb CLI o programación)
# Desde dentro del contenedor, Dockerd expone el estado via API:
$ curl --unix-socket /var/run/docker.sock http://localhost/networks | jq .
```

Tras un reinicio del daemon de Docker, libnetwork lee este archivo para reconstruir el estado de todas las redes. Si el archivo se corrompe o se pierde, las redes desaparecen (aunque los contenedores sigan existiendo).

---

## 6.2 Bridge Network: La Red por Defecto

### 6.2.1 ¿Qué es un bridge de Linux?

Un **bridge** en Linux es un switch virtual de capa 2. Conmuta tramas Ethernet entre las interfaces que están conectadas a él, igual que un switch físico. Aprende direcciones MAC, reenvía tramas y puede participar en STP (Spanning Tree Protocol).

Cuando instalas Docker, el daemon crea automáticamente un bridge llamado `docker0` en el host:

```bash
$ ip addr show docker0
3: docker0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default
    link/ether 02:42:ac:11:00:01 brd ff:ff:ff:ff:ff:ff
    inet 172.17.0.1/16 brd 172.17.255.255 scope global docker0
       valid_lft forever preferred_lft forever
    inet6 fe80::42:acff:fe11:1/64 scope link
       valid_lft forever preferred_lft forever
```

Observa:
- IP `172.17.0.1/16`: esta es la puerta de enlace (gateway) para todos los contenedores conectados a este bridge.
- Subred `172.17.0.0/16`: Docker asigna IPs del rango `172.17.0.2` a `172.17.255.254` a los contenedores.
- MAC `02:42:ac:11:00:01`: Docker usa MACs predecibles (basadas en la IP) para facilitar debugging.

Docker también configura reglas de iptables para proporcionar conectividad externa (SNAT/MASQUERADE) a los contenedores conectados a `docker0`.

### 6.2.2 La red bridge por defecto

Cada instalación de Docker incluye tres redes predefinidas:

```bash
$ docker network ls
NETWORK ID          NAME                DRIVER              SCOPE
a1b2c3d4e5f6        bridge              bridge              local
b2c3d4e5f6a1        host                host                local
c3d4e5f6a1b2        none                null                local
```

La red `bridge` (nombre) es la red por defecto. Si ejecutas `docker run` sin especificar `--network`, el contenedor se conecta automáticamente a esta red:

```bash
$ docker run -d --name cont1 nginx
$ docker run -d --name cont2 nginx
```

Ambos contenedores están ahora en la red `bridge`.

### 6.2.3 Inspeccionando la red bridge

```bash
$ docker network inspect bridge
[
    {
        "Name": "bridge",
        "Id": "a1b2c3d4e5f67890abcdef1234567890abcdef1234567890abcdef12345678",
        "Created": "2024-01-15T10:30:00.123456789Z",
        "Scope": "local",
        "Driver": "bridge",
        "EnableIPv6": false,
        "IPAM": {
            "Driver": "default",
            "Options": null,
            "Config": [
                {
                    "Subnet": "172.17.0.0/16",
                    "Gateway": "172.17.0.1"
                }
            ]
        },
        "Internal": false,
        "Attachable": false,
        "Ingress": false,
        "ConfigFrom": {
            "Network": ""
        },
        "ConfigOnly": false,
        "Containers": {
            "abc123def456...": {
                "Name": "cont1",
                "EndpointID": "endpoint1id...",
                "MacAddress": "02:42:ac:11:00:02",
                "IPv4Address": "172.17.0.2/16",
                "IPv6Address": ""
            },
            "def789abc012...": {
                "Name": "cont2",
                "EndpointID": "endpoint2id...",
                "MacAddress": "02:42:ac:11:00:03",
                "IPv4Address": "172.17.0.3/16",
                "IPv6Address": ""
            }
        },
        "Options": {
            "com.docker.network.bridge.default_bridge": "true",
            "com.docker.network.bridge.enable_icc": "true",
            "com.docker.network.bridge.enable_ip_masquerade": "true",
            "com.docker.network.bridge.host_binding_ipv4": "0.0.0.0",
            "com.docker.network.bridge.name": "docker0",
            "com.docker.network.driver.mtu": "1500"
        },
        "Labels": {}
    }
]
```

Analicemos las partes clave de este output:

**IPAM.Config:**
- `Subnet: "172.17.0.0/16"`: La subred completa. 65534 direcciones disponibles.
- `Gateway: "172.17.0.1"`: El bridge `docker0` en el host. Es el default gateway para los contenedores.

**Containers:**
- Cada contenedor tiene una IP única en la subred.
- La MAC está derivada de la IP (prefijo `02:42` + IP en hex).

**Options:**
- `com.docker.network.bridge.enable_icc: "true"`: ICC = Inter-Container Communication. Los contenedores en el mismo bridge pueden comunicarse entre sí. Si se pone a `false`, solo pueden hablar con el exterior pero no entre ellos.
- `com.docker.network.bridge.enable_ip_masquerade: "true"`: SNAT/MASQUERADE habilitado. Los contenedores pueden salir a internet.
- `com.docker.network.bridge.host_binding_ipv4: "0.0.0.0"`: Los puertos publicados (-p) escuchan en todas las interfaces del host por defecto.

### 6.2.4 Port Forwarding: cómo funciona -p

Cuando ejecutas:

```bash
$ docker run -d -p 8080:80 --name web nginx
```

Docker programa dos reglas fundamentales en iptables. Vamos a verlas:

#### Regla DNAT (entrada)

El tráfico que llega al puerto 8080 del host se redirige al puerto 80 del contenedor:

```
Regla en la cadena DOCKER (tabla nat, PREROUTING):

PKT: host:8080  -->  DNAT: 172.17.0.2:80
```

```bash
$ sudo iptables -t nat -L DOCKER -n
Chain DOCKER (2 references)
target     prot opt source               destination
DNAT       tcp  --  0.0.0.0/0            0.0.0.0/0            tcp dpt:8080 to:172.17.0.2:80
```

Esta regla dice: cualquier paquete TCP destinado al puerto 8080 en cualquier interfaz, cámbiale el destino a `172.17.0.2:80`.

#### Docker proxy (modo userspace)

En algunas configuraciones, Docker también levanta un proxy en userspace (`docker-proxy`) que escucha en el puerto del host y reenvía al contenedor:

```bash
$ sudo ss -tlnp | grep 8080
LISTEN  0  128  0.0.0.0:8080  0.0.0.0:*  users:(("docker-proxy",pid=1234,fd=4))
```

El `docker-proxy` existe como fallback cuando las reglas de iptables no pueden manejar cierto tráfico (por ejemplo, conexiones desde el mismo host que van a un puerto mapeado sin passar por PREROUTING). Puedes deshabilitarlo con `--userland-proxy=false` en el daemon para mejor rendimiento.

#### Regla MASQUERADE (salida)

Cuando un contenedor envía tráfico al exterior (internet), Docker SNATea la IP de origen para que parezca venir del host:

```
Regla en la cadena POSTROUTING (tabla nat):

PKT: 172.17.0.2:XXXXX  -->  SNAT/MASQUERADE: 192.168.1.100:YYYYY
(pasa a usar la IP del host como origen)
```

```bash
$ sudo iptables -t nat -L POSTROUTING -n
Chain POSTROUTING (policy ACCEPT)
target     prot opt source               destination
MASQUERADE  all  --  172.17.0.0/16        0.0.0.0/0
```

Esto explica por qué, desde fuera, las conexiones de tus contenedores parecen originarse en el host.

### 6.2.5 docker port

Para ver los mapeos de puertos de un contenedor en ejecución:

```bash
$ docker port web
80/tcp -> 0.0.0.0:8080
80/tcp -> [::]:8080
```

También puedes preguntar por un puerto específico:

```bash
$ docker port web 80
0.0.0.0:8080
[::]:8080
```

### 6.2.6 Limitaciones del bridge por defecto

La red `bridge` por defecto es práctica para empezar, pero tiene limitaciones importantes que la hacen inadecuada para entornos de producción:

#### 1. Sin resolución DNS automática

En el bridge por defecto, los contenedores solo pueden comunicarse por **dirección IP**, no por nombre de contenedor:

```bash
$ docker run -d --name app nginx
$ docker run --rm alpine ping app
ping: bad address 'app'
```

El nombre `app` no se resuelve. Debes usar la IP:

```bash
$ docker run --rm alpine ping 172.17.0.2
PING 172.17.0.2 (172.17.0.2): 56 data bytes
64 bytes from 172.17.0.2: seq=0 ttl=64 time=0.120 ms
```

Esto hace que las aplicaciones tengan que hardcodear IPs o implementar su propio descubrimiento de servicios.

#### 2. Sin aislamiento entre proyectos

Todos los contenedores conectados al bridge por defecto comparten el mismo espacio de red. Si ejecutas `docker run` sin `--network`, todos caen en el mismo bridge. No hay separación lógica entre proyectos distintos.

#### 3. Sin reconexión en caliente

No puedes desconectar un contenedor del bridge por defecto y conectarlo a otra red mientras está corriendo:

```bash
$ docker network disconnect bridge cont1
Error response from daemon: container cont1 failed to leave network bridge:
  cannot disconnect a container from the default bridge network
```

Tendrías que parar el contenedor, eliminarlo y recrearlo con otra red. Esto es un problema en entornos dinámicos.

#### 4. Compartición de variables de entorno (--link, obsoleto)

En el pasado, Docker ofrecía `--link` para que los contenedores en el bridge por defecto pudieran comunicarse por nombre (inyectando variables de entorno y entradas en `/etc/hosts`). Esto creaba dependencias ocultas, problemas de orden de arranque y era frágil. `--link` está oficialmente obsoleto y **no debe usarse**.

---

## 6.3 Redes Definidas por el Usuario

Las redes definidas por el usuario resuelven todas las limitaciones del bridge por defecto. Son el mecanismo recomendado para cualquier entorno que no sea un simple test.

### 6.3.1 Crear una red de usuario

```bash
$ docker network create mi_red
a8f3d9c1b2e4567890abcdef1234567890abcdef1234567890abcdef12345
```

Docker crea un nuevo bridge (con un nombre como `br-a8f3d9c1b2e4`) en el host y una nueva subred (por defecto, del rango `172.18.0.0/16`, luego `172.19.0.0/16`, etc.):

```bash
$ ip addr show br-a8f3d9c1b2e4
5: br-a8f3d9c1b2e4: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default
    link/ether 02:42:ac:12:00:01 brd ff:ff:ff:ff:ff:ff
    inet 172.18.0.1/16 brd 172.18.255.255 scope global br-a8f3d9c1b2e4
       valid_lft forever preferred_lft forever
```

### 6.3.2 Ventajas fundamentales

#### Ventaja 1: DNS automático

Esta es la killer feature. Los contenedores en una red definida por el usuario pueden resolverse **por nombre** automáticamente:

```bash
$ docker run -d --name app1 --network mi_red nginx
$ docker run --rm --network mi_red alpine ping app1
PING app1 (172.18.0.2): 56 data bytes
64 bytes from 172.18.0.2: seq=0 ttl=64 time=0.098 ms
64 bytes from 172.18.0.2: seq=1 ttl=64 time=0.105 ms
```

El nombre del contenedor (`app1`) se resuelve a su IP automáticamente. Docker mantiene un servidor DNS embebido en `127.0.0.11` dentro de cada contenedor. Lo veremos en detalle en la sección 6.8.

Esto permite patrones como:

```bash
# Base de datos
docker run -d --name db --network mi_red mysql:8

# Aplicación que se conecta a "db" como hostname
docker run -d --name api --network mi_red -e DB_HOST=db mi_api:latest
```

La aplicación solo necesita saber el nombre `db`. Ni una sola IP hardcodeada.

#### Ventaja 2: Mejor aislamiento

Solo los contenedores explícitamente conectados a `mi_red` pueden verse entre sí:

```bash
$ docker run -d --name aislado nginx
$ docker run --rm --network mi_red alpine ping aislado
ping: bad address 'aislado'
```

El contenedor `aislado` (en la red bridge por defecto) es invisible desde `mi_red`. Cada red es un dominio de broadcast separado.

#### Ventaja 3: Conexión/desconexión en caliente

Puedes conectar y desconectar contenedores de la red mientras están en ejecución:

```bash
$ docker run -d --name dinamico nginx
$ docker network connect mi_red dinamico
$ docker network inspect mi_red  # ahora aparece "dinamico"

$ docker network disconnect mi_red dinamico
$ docker network inspect mi_red  # ya no aparece
```

Esto permite arquitecturas dinámicas donde los servicios se unen o abandonan redes según necesidad.

### 6.3.3 Listando e inspeccionando redes

```bash
$ docker network ls
NETWORK ID          NAME                DRIVER              SCOPE
a8f3d9c1b2e4        mi_red              bridge              local
a1b2c3d4e5f6        bridge              bridge              local
b2c3d4e5f6a1        host                host                local
c3d4e5f6a1b2        none                null                local
```

```bash
$ docker network inspect mi_red
[
    {
        "Name": "mi_red",
        "Id": "a8f3d9c1b2e456...",
        "Scope": "local",
        "Driver": "bridge",
        "EnableIPv6": false,
        "IPAM": {
            "Driver": "default",
            "Config": [
                {
                    "Subnet": "172.18.0.0/16",
                    "Gateway": "172.18.0.1"
                }
            ]
        },
        "Containers": {},
        "Options": {},
        "Labels": {}
    }
]
```

### 6.3.4 docker run --network

Para lanzar un contenedor directamente en una red:

```bash
$ docker run -d --name web --network mi_red nginx
```

Puedes conectar un contenedor a múltiples redes:

```bash
$ docker network create frontend
$ docker network create backend

$ docker run -d --name app --network frontend nginx
$ docker network connect backend app
```

Ahora `app` tiene interfaces en ambas redes:

```bash
$ docker exec app ip addr
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN
2: tunl0@NONE: <NOARP> mtu 1480 qdisc noop state DOWN
3: eth0@if10: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP
    link/ether 02:42:ac:12:00:02 brd ff:ff:ff:ff:ff:ff
    inet 172.18.0.2/16 brd 172.18.255.255 scope global eth0
4: eth1@if12: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP
    link/ether 02:42:ac:13:00:02 brd ff:ff:ff:ff:ff:ff
    inet 172.19.0.2/16 brd 172.19.255.255 scope global eth1
```

### 6.3.5 Subred y gateway personalizados

Por defecto, Docker elige una subred del bloque `172.16.0.0/12` automáticamente (172.17.0.0/16, 172.18.0.0/16, etc). Para tener control total, especifica `--subnet` y `--gateway`:

```bash
$ docker network create \
  --subnet=10.10.0.0/16 \
  --gateway=10.10.0.1 \
  mi_red_personalizada
```

```bash
$ docker run --rm --network mi_red_personalizada alpine ip route
default via 10.10.0.1 dev eth0
10.10.0.0/16 dev eth0 scope link  src 10.10.0.2
```

Esto es esencial cuando:
- Necesitas que las IPs de los contenedores no colisionen con redes existentes (VPN, oficina).
- Integras Docker con sistemas externos que esperan IPs en rangos específicos.
- Quieres documentar claramente la asignación de subredes.

### 6.3.6 IPAM avanzado: --ip-range y --ip

#### --ip-range: Restringir el rango de IPs asignables

```bash
$ docker network create \
  --subnet=10.10.0.0/16 \
  --ip-range=10.10.5.0/24 \
  --gateway=10.10.0.1 \
  red_restringida
```

Los contenedores solo recibirán IPs del rango `10.10.5.0/24`. El resto de la subred (`10.10.0.0/24` hasta `10.10.4.0/24` y `10.10.6.0/24` en adelante) está reservado. Esto es útil cuando tienes IPs fijas para equipos físicos o VMs en la misma subred y quieres evitar colisiones.

#### --ip: Asignar IP fija a un contenedor

```bash
$ docker run -d --name db \
  --network red_restringida \
  --ip 10.10.5.100 \
  mysql:8
```

La IP `10.10.5.100` se asigna estáticamente a `db`. El contenedor siempre tendrá esa IP (incluso tras reinicios). Otros servicios pueden hardcodear `10.10.5.100` para conectarse a la base de datos (aunque el DNS es mejor práctica).

### 6.3.7 Red interna (--internal)

Una red `--internal` aísla completamente a los contenedores del mundo exterior. No pueden acceder a internet ni recibir tráfico externo:

```bash
$ docker network create --internal red_interna
```

```bash
$ docker run --rm --network red_interna alpine ping 8.8.8.8
PING 8.8.8.8 (8.8.8.8): 56 data bytes
^C
--- 8.8.8.8 ping statistics ---
3 packets transmitted, 0 packets received, 100% packet loss
```

Sin embargo, los contenedores dentro de `red_interna` sí pueden comunicarse entre sí normalmente.

Casos de uso:
- Base de datos que solo debe ser accesible por la aplicación (y nunca desde internet).
- Cola de mensajes interna (RabbitMQ, Kafka) en un backend privado.
- Microservicios de procesamiento que no necesitan acceso externo.

### 6.3.8 --attachable (Swarm)

En Docker Swarm, las redes overlay son el mecanismo estándar. Pero a veces necesitas que un contenedor standalone (no servicio Swarm) se conecte manualmente a una red overlay. Para eso sirve `--attachable`:

```bash
$ docker network create --driver overlay --attachable mi_overlay_attachable
```

Luego, un contenedor normal puede conectarse:

```bash
$ docker run --rm --network mi_overlay_attachable alpine ping servicio_swarm
```

Sin `--attachable`, solo los servicios Swarm definidos con `--network` pueden usar la red.

### 6.3.9 --scope: local vs swarm

- **local**: La red solo existe en el nodo Docker actual. Es el valor por defecto para bridge, host, macvlan.
- **swarm**: La red existe a nivel de clúster Swarm. Overlay networks siempre son scope swarm. Bridge networks no pueden ser scope swarm.

```bash
$ docker network inspect mi_red | grep Scope
        "Scope": "local",
```

Las redes swarm son visibles desde cualquier manager:

```bash
$ docker network ls --filter scope=swarm
NETWORK ID          NAME                DRIVER              SCOPE
xyz123abc456        ingress             overlay             swarm
xyz456def789        mi_overlay          overlay             swarm
```

### 6.3.10 Opciones adicionales del driver bridge

Al crear un bridge, puedes pasar opciones al driver con `-o` (o `--opt`):

```bash
$ docker network create \
  -o com.docker.network.bridge.name=mi_bridge \
  -o com.docker.network.bridge.enable_icc=false \
  -o com.docker.network.bridge.enable_ip_masquerade=false \
  -o com.docker.network.bridge.host_binding_ipv4=127.0.0.1 \
  -o com.docker.network.driver.mtu=1400 \
  red_opciones
```

| Opción | Descripción | Default |
|--------|-------------|---------|
| `com.docker.network.bridge.name` | Nombre del bridge Linux | `br-<id>` |
| `com.docker.network.bridge.enable_icc` | Comunicación entre contenedores | `true` |
| `com.docker.network.bridge.enable_ip_masquerade` | NAT de salida | `true` |
| `com.docker.network.bridge.host_binding_ipv4` | IP de escucha para `-p` | `0.0.0.0` |
| `com.docker.network.driver.mtu` | MTU de la red | `1500` |

---

## 6.4 Host Network: Rendimiento Máximo

### 6.4.1 Concepto

El driver `host` elimina completamente el aislamiento de red del contenedor. El contenedor comparte la pila de red del host directamente: mismo namespace de red, mismas interfaces, mismas tablas de enrutamiento, mismos puertos.

```
+----------------------------------------------------------+
|                    HOST (namespace raíz)                  |
|                                                          |
|   eth0: 192.168.1.100                                    |
|   docker0: 172.17.0.1                                    |
|   lo: 127.0.0.1                                          |
|                                                          |
|   +-------------------+     +--------------------------+ |
|   |   Contenedor A    |     |     Contenedor B         | |
|   |  (--network host) |     |  (--network host)        | |
|   |                   |     |                          | |
|   |  eth0 = 192.168.  |     |  eth0 = 192.168.         | |
|   |         1.100     |     |         1.100            | |
|   +-------------------+     +--------------------------+ |
|                                                          |
|   ¡Ambos comparten las mismas interfaces y puertos!      |
+----------------------------------------------------------+
```

### 6.4.2 Uso

```bash
$ docker run --rm --network host alpine ip addr
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP
    link/ether aa:bb:cc:dd:ee:ff
    inet 192.168.1.100/24 brd 192.168.1.255 scope global dynamic eth0
3: docker0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP
    inet 172.17.0.1/16 brd 172.17.255.255 scope global docker0
```

El contenedor ve **exactamente** las mismas interfaces que el host. Si el host tiene la IP `192.168.1.100`, el contenedor también. No hay `eth0@ifXX` virtual, no hay NAT, no hay bridge intermedio.

### 6.4.3 Ventajas

**Rendimiento**: Sin capas de virtualización de red. El tráfico no atraviesa bridges, no se aplica NAT, no hay pares veth. El throughput es prácticamente el del hardware. Para aplicaciones de alto rendimiento (bases de datos, sistemas de caché, proxies de alto tráfico), `host` puede ser la diferencia entre latencia de microsegundos y milisegundos.

**Simplicidad**: No necesitas preocuparte por mapeo de puertos. Si la aplicación escucha en el puerto 80 del contenedor, escucha en el puerto 80 del host. `-p` no tiene sentido en modo host y Docker lo ignora.

### 6.4.4 Desventajas

**Conflictos de puertos**: Si dos contenedores usan `--network host` y ambos intentan escuchar en el puerto 80, el segundo fallará:

```bash
$ docker run -d --name nginx1 --network host nginx  # ok
$ docker run -d --name nginx2 --network host nginx  # falla, puerto 80 ocupado
Error: port is already allocated
```

**Sin aislamiento**: El contenedor puede ver todo el tráfico del host. Puede interferir con las reglas de iptables. Un proceso malicioso podría hacer ARP spoofing en la red del host.

**Sin DNS automático**: Como no hay un bridge de Docker gestionando el DNS, los nombres de contenedor no se resuelven automáticamente.

### 6.4.5 Casos de uso

- **HAProxy/Nginx como balanceador de carga**: Para manejar decenas de miles de conexiones por segundo sin que el bridge sea cuello de botella.
- **Redis/Memcached**: Cachés que requieren latencia mínima.
- **Node Exporter de Prometheus o cAdvisor**: Herramientas que necesitan acceder a métricas del host (`/proc`, `/sys`).
- **Aplicaciones que abren puertos efímeros masivamente** (FTP pasivo, SIP/RTP para VoIP): Para evitar la complejidad de mapear rangos de puertos.

---

## 6.5 Overlay Network: Comunicación Multi-Host (Swarm)

### 6.5.1 El problema del multi-host

Hasta ahora, todo lo que hemos visto funciona en un único host Docker. Pero en producción necesitas que contenedores en diferentes máquinas se comuniquen como si estuvieran en la misma red local. Ahí entran las redes overlay.

```
+---------------------------+           +---------------------------+
|         HOST 1             |           |         HOST 2             |
|                           |           |                           |
|  Contenedor A             |           |  Contenedor B             |
|  10.0.0.2                 |           |  10.0.0.3                 |
|                           |           |                           |
|      |                    |           |      |                    |
|  +---+----+               |           |  +---+----+               |
|  |  VXLAN  |==============|=============|==|  VXLAN  |            |
|  | Tunnel  |   UDP:4789   |           |  | Tunnel  |            |
|  +--------+               |           |  +--------+               |
|                           |           |                           |
|  eth0: 192.168.1.10       |           |  eth0: 192.168.1.20      |
+---------------------------+           +---------------------------+
              |                                      |
       [RED FÍSICA - Ethernet / IP]
```

### 6.5.2 VXLAN: Virtual Extensible LAN

El driver overlay de Docker utiliza **VXLAN** (Virtual Extensible LAN, RFC 7348) para crear túneles L2 sobre redes L3.

**¿Cómo funciona?**

1. Cuando el Contenedor A (10.0.0.2) quiere enviar un paquete al Contenedor B (10.0.0.3), el tráfico sale de su interfaz `eth0` (veth) hacia el bridge overlay del Host 1.
2. El bridge determina que 10.0.0.3 está en el Host 2 (192.168.1.20).
3. Se encapsula la trama Ethernet original dentro de un paquete UDP con destino `192.168.1.20:4789`.
4. El paquete UDP viaja por la red física subyacente (underlay).
5. En el Host 2, el endpoint VXLAN desencapsula la trama original y la entrega al Contenedor B.

```
+----------------------------------------------------------+
|                  PAQUETE VXLAN (UDP)                      |
|                                                          |
|  +------------------+-----------------------------------+ |
|  | Cabecera Externa | Payload VXLAN                     | |
|  | (IP Underlay)    |                                   | |
|  |                  |  +-------+-----------------------+ | |
|  | SRC: 192.168.1.10|  |VXLAN  | Trama Ethernet Orig. | | |
|  | DST: 192.168.1.20|  |Header |                       | | |
|  | UDP: 4789        |  |(VNI)  | SRC MAC: aa:bb:cc... | | |
|  +------------------+  |       | DST MAC: dd:ee:ff... | | |
|                        |       | SRC IP: 10.0.0.2     | | |
|                        |       | DST IP: 10.0.0.3     | | |
|                        |       | Payload: HTTP GET... | | |
|                        +-------+-----------------------+ | |
+----------------------------------------------------------+
```

**VNI (VXLAN Network Identifier)**: Un ID de 24 bits que identifica la red overlay. Permite hasta 16 millones de redes overlay diferentes en la misma infraestructura física. Docker asigna un VNI por cada red overlay.

### 6.5.3 Requisitos para overlay

Las redes overlay requieren que los nodos Docker estén en modo Swarm (recomendado) o que uses un almacén clave-valor externo (obsoleto y no recomendado).

**Con Swarm (recomendado):**

```bash
# Inicializar Swarm (en el manager)
$ docker swarm init --advertise-addr 192.168.1.10
Swarm initialized: current node (abc123...) is now a manager.

# En los workers
$ docker swarm join --token SWMTKN-1-... 192.168.1.10:2377
```

**Requisitos de red para Swarm:**
- Puerto **2377/tcp**: Comunicación de gestión del clúster.
- Puerto **7946/tcp y 7946/udp**: Comunicación de nodos (gossip protocol, Serf).
- Puerto **4789/udp**: Tráfico de overlay (VXLAN).

```bash
# Abrir puertos en el firewall de cada nodo
$ sudo ufw allow 2377/tcp
$ sudo ufw allow 7946/tcp
$ sudo ufw allow 7946/udp
$ sudo ufw allow 4789/udp
```

### 6.5.4 Crear una red overlay

```bash
$ docker network create \
  --driver overlay \
  --subnet 10.0.0.0/24 \
  --gateway 10.0.0.1 \
  mi_overlay
```

```bash
$ docker network ls
NETWORK ID          NAME                DRIVER              SCOPE
abc123def456        mi_overlay          overlay             swarm
```

Ahora puedes crear servicios que usen esta red:

```bash
$ docker service create \
  --name web \
  --network mi_overlay \
  --replicas 3 \
  nginx
```

Los 3 contenedores (réplicas) pueden estar en 3 nodos diferentes, pero todos comparten la subred `10.0.0.0/24` y pueden comunicarse por IP y por nombre (DNS de Swarm).

### 6.5.5 Encriptación IPSec (--opt encrypted)

Por defecto, el tráfico VXLAN no está encriptado. Cualquiera con acceso a la red física puede capturar los paquetes y leer el payload (aunque el contenido esté en una VXLAN).

Para entornos donde la seguridad es crítica, Docker ofrece encriptación IPSec a nivel de overlay:

```bash
$ docker network create \
  --driver overlay \
  --opt encrypted \
  mi_overlay_segura
```

Con `--opt encrypted`, Docker establece túneles IPSec (usando el subsistema IPSec del kernel de Linux, no userspace) entre todos los pares de nodos que participan en la red overlay. Cada paquete VXLAN se encripta antes de ser enviado.

**Consideraciones de rendimiento con encriptación:**
- Latencia adicional por el cifrado/descifrado (típicamente 10-30%).
- Consumo de CPU (el cifrado AES-NI por hardware mitiga esto).
- Throughput reducido en cargas I/O intensivas.

Para la mayoría de cargas de trabajo en un centro de datos privado, la encriptación es recomendable y el overhead es aceptable.

### 6.5.6 La red ingress de Swarm

Cuando inicializas Swarm, Docker crea automáticamente una red overlay especial llamada `ingress`:

```bash
$ docker network ls --filter name=ingress
NETWORK ID          NAME                DRIVER              SCOPE
ingress_id          ingress             overlay             swarm
```

La red `ingress` es la responsable del **routing mesh**: cuando publicas un puerto en un servicio (`--publish 80:80`), cualquier nodo del clúster, reciba o no una réplica del servicio, acepta la conexión en el puerto 80 y la reenvía a un contenedor que sí tenga la réplica.

```
          Cliente
             |
     (petición HTTP a :80)
             |
    +--------+--------+
    |                  |
 Nodo 1               Nodo 2
 (sin réplica)        (con réplica web)
    |                      |
 puerto 80         <--- routing mesh --->   contenedor web:80
 (escucha)                                   (sirve la petición)
```

Esto permite balanceo de carga transparente sin necesidad de un balanceador externo.

### 6.5.7 Limitaciones de overlay

- **Latencia**: El encapsulamiento VXLAN añade overhead. En tests de laboratorio se observan entre 50-200 microsegundos adicionales por paquete respecto a comunicación directa.
- **MTU**: Los paquetes encapsulados son más grandes. La MTU por defecto de las interfaces overlay es 1450 (vs 1500 estándar). Si tu red física no soporta jumbo frames, puede haber fragmentación.
- **Debugging complejo**: Los problemas de overlay pueden ser difíciles de diagnosticar porque involucran dos capas de red (underlay + overlay).
- **No es para todas las cargas**: Para tráfico de alta velocidad (10Gbps+), considera macvlan o soluciones como Calico/Flannel en modo VXLAN con offloading de hardware.

### 6.5.8 DNS en Swarm: VIP vs DNSRR

Cuando creas un servicio en Swarm conectado a una red overlay, Docker le asigna automáticamente una **VIP (Virtual IP)**. La VIP es una IP estable que actúa como balanceador de carga interno. Cuando otro contenedor resuelve el nombre del servicio, recibe la VIP, y el tráfico se balancea a las réplicas sanas:

```
                  DNS: web -> 10.0.0.100 (VIP)
                             |
              +--------------+--------------+
              |              |              |
         Réplica 1      Réplica 2      Réplica 3
        10.0.0.2       10.0.0.3       10.0.0.5
         (Nodo 1)       (Nodo 2)       (Nodo 3)
```

**Ventajas de VIP:**
- Balanceo de carga L4 transparente (IPVS en el kernel de Linux, no userspace).
- Health checking automático: si una réplica falla, se saca del pool.
- Persistencia de sesión configurable.
- Una sola IP estable para el servicio, independientemente de cuántas réplicas haya.

**Modo DNSRR (DNS Round-Robin):**

Swarm también ofrece un modo alternativo donde el DNS devuelve directamente las IPs de las réplicas en lugar de una VIP:

```bash
$ docker service create --name web --network mi_overlay \
  --endpoint-mode dnsrr \
  --replicas 3 nginx
```

En modo `dnsrr`, cuando un cliente resuelve `web`, obtiene las 3 IPs reales de los contenedores. El cliente es responsable de elegir una y balancear.

**VIP vs DNSRR: cuándo usar cada uno:**

| Característica | VIP | DNSRR |
|---|---|---|
| Balanceo | L4, kernel (IPVS) | L7, cliente decide |
| Latencia | Mínima (IPVS) | Resolución DNS adicional |
| Conexiones largas | Estables (sticky opcional) | No hay afinidad |
| Aplicaciones sin balanceador | Funciona | Requiere lógica en el cliente |
| Estadísticas | Por VIP | Solo DNS logs |

### 6.5.9 Servicios multi-red en Swarm

Los servicios pueden conectarse a múltiples redes overlay, igual que los contenedores locales a múltiples bridges. Este es el patrón de microservicios en Swarm:

```bash
$ docker network create --driver overlay frontend
$ docker network create --driver overlay backend --internal

$ docker service create --name api \
  --network frontend \
  --network backend \
  --replicas 5 \
  mi_api:latest

$ docker service create --name db \
  --network backend \
  mysql:8
```

El servicio `api` está en ambas redes overlay, creando el mismo patrón de "puente" que vimos con bridges locales. Este patrón escala a docenas de servicios sin cambiar la lógica de red.

---

## 6.6 Macvlan / Ipvlan: Contenedores con Identidad de Red

### 6.6.1 El concepto

Macvlan e Ipvlan permiten que los contenedores obtengan **su propia dirección MAC e IP en la red física**. Para la red externa, cada contenedor es indistinguible de una máquina física o VM.

```
+----------------------------------------------------------+
|                      RED FÍSICA                          |
|             192.168.1.0/24 (VLAN 100)                    |
|                                                          |
|   +--------------+  +--------------+  +--------------+   |
|   |   Servidor   |  |   Host       |  |  Contenedor  |   |
|   |   Físico     |  |   Docker     |  |  A (macvlan) |   |
|   |              |  |              |  |              |   |
|   | MAC: aa:bb   |  | MAC: cc:dd   |  | MAC: ee:ff   |   |
|   | IP: .10      |  | IP: .100     |  | IP: .200     |   |
|   +--------------+  +--------------+  +--------------+   |
|                          |                               |
|                          |  +--------------+             |
|                          +--+  Contenedor  |             |
|                             |  B (macvlan) |             |
|                             |              |             |
|                             | MAC: gg:hh   |             |
|                             | IP: .201     |             |
|                             +--------------+             |
+----------------------------------------------------------+
```

Cada contenedor aparece como un dispositivo físico independiente en la red.

### 6.6.2 Crear una red macvlan

```bash
$ docker network create \
  --driver macvlan \
  --subnet=192.168.1.0/24 \
  --gateway=192.168.1.1 \
  --ip-range=192.168.1.200/29 \
  -o parent=eth0 \
  mi_macvlan
```

- `--driver macvlan`: Usa el driver macvlan.
- `--subnet`: La subred física real donde los contenedores obtendrán IPs.
- `--gateway`: El gateway de la red física (normalmente tu router).
- `-o parent=eth0`: La interfaz física del host a la que se "engancha" macvlan.
- `--ip-range`: Rango de IPs que Docker puede asignar automáticamente (opcional).

### 6.6.3 Crear contenedores con macvlan

```bash
$ docker run -d --name app1 --network mi_macvlan --ip 192.168.1.200 nginx
$ docker run -d --name app2 --network mi_macvlan --ip 192.168.1.201 nginx
```

Desde cualquier máquina en la red `192.168.1.0/24`:

```bash
$ ping 192.168.1.200
PING 192.168.1.200 (192.168.1.200): 56 data bytes
64 bytes from 192.168.1.200: icmp_seq=0 ttl=64 time=0.320 ms

$ curl http://192.168.1.200
<!DOCTYPE html>
<html>
...
```

Los contenedores responden directamente, sin NAT, sin port forwarding. Exactamente como si fueran dos máquinas físicas.

### 6.6.4 Modos de macvlan

Macvlan tiene varios modos. Docker soporta principalmente dos:

#### Modo bridge (por defecto)

Los contenedores pueden comunicarse entre sí a través de la interfaz padre y con la red externa. El tráfico entre contenedores en el mismo host se filtra en el kernel y no sale a la red física.

#### Modo 802.1q (VLAN trunk)

Puedes crear subinterfaces VLAN en el host y asignar cada una a una red macvlan diferente:

```bash
# Crear subinterfaces VLAN en el host
$ sudo ip link add link eth0 name eth0.100 type vlan id 100
$ sudo ip link add link eth0 name eth0.200 type vlan id 200
$ sudo ip link set eth0.100 up
$ sudo ip link set eth0.200 up

# Red macvlan en VLAN 100
$ docker network create \
  -d macvlan \
  --subnet=10.100.0.0/24 \
  --gateway=10.100.0.1 \
  -o parent=eth0.100 \
  vlan100

# Red macvlan en VLAN 200
$ docker network create \
  -d macvlan \
  --subnet=10.200.0.0/24 \
  --gateway=10.200.0.1 \
  -o parent=eth0.200 \
  vlan200
```

Esto permite segregación de tráfico a nivel de VLAN para contenedores, ideal en entornos enterprise que ya usan VLANs para separar departamentos, entornos (dev/staging/prod) o niveles de seguridad.

### 6.6.5 Ventajas de macvlan

- **Rendimiento**: Sin NAT, sin bridge virtual, sin encapsulamiento overlay. El tráfico usa directamente la interfaz física. En benchmarks, macvlan iguala el rendimiento de aplicaciones nativas.
- **Integración con sistemas legacy**: Dispositivos físicos, firewalls, balanceadores legacy pueden comunicarse con contenedores sin saber que lo son.
- **VLANs**: Encaja perfectamente en redes corporativas segmentadas por VLAN.
- **IPs estáticas en la red corporativa**: DHCP del router físico puede asignar IPs (si configuras DHCP manualmente o usas ip-range).

### 6.6.6 Desventajas de macvlan

- **Modo promiscuo**: Muchos entornos cloud (AWS, Azure, GCP) bloquean el tráfico con MACs desconocidas. Necesitas que el hipervisor permita MAC spoofing o modo promiscuo. Esto hace que macvlan sea **imposible en la mayoría de nubes públicas** sin configuración especial del hipervisor.
- **Sin DHCP automático**: Docker no tiene un cliente DHCP para macvlan. Debes asignar IPs manualmente o gestionarlas externamente.
- **El host no puede comunicarse con sus contenedores macvlan**: Esta es una limitación del kernel de Linux. El tráfico entre el namespace raíz y un namespace hijo vía macvlan se descarta por diseño. Solución: crear un macvlan adicional en el host:

```bash
$ sudo ip link add macvlan_host link eth0 type macvlan mode bridge
$ sudo ip addr add 192.168.1.250/24 dev macvlan_host
$ sudo ip link set macvlan_host up
```

Ahora el host puede alcanzar los contenedores vía la dirección `192.168.1.250`.

### 6.6.7 Ipvlan: la alternativa ligera

Ipvlan es similar a macvlan pero todos los contenedores comparten la misma dirección MAC del host. La diferenciación se hace a nivel de IP.

```bash
$ docker network create \
  --driver ipvlan \
  --subnet=192.168.1.0/24 \
  --gateway=192.168.1.1 \
  -o parent=eth0 \
  -o ipvlan_mode=l2 \
  mi_ipvlan
```

**Modos de ipvlan:**

- **L2**: Opera como un switch L2. El tráfico se conmuta por MAC (compartida) + IP. Es el modo por defecto.
- **L3**: Opera como un router. El host actúa como router entre la red ipvlan y la red física. No hay flooding ARP. Más eficiente pero requiere configuración de rutas en el router físico.
- **L3S**: Similar a L3 pero con soporte para iptables/netfilter dentro del contenedor (necesario para Kubernetes NetworkPolicy, por ejemplo).

**Ventaja de ipvlan sobre macvlan:**
- Menor consumo de recursos en el switch físico (una sola MAC para cientos de contenedores).
- Mejor compatibilidad con entornos cloud que limitan el número de MACs por puerto.

**Desventaja:**
- El modo L3 requiere configuración de enrutamiento adicional.

### 6.6.8 Macvlan vs Ipvlan: cuándo usar cada uno

| Criterio | Macvlan | Ipvlan |
|----------|---------|--------|
| Cada contenedor tiene MAC única | Sí | No (comparten MAC del host) |
| Compatibilidad cloud | Baja | Media (L3 puede funcionar) |
| Aislamiento L2 | Completo | Comparten dominio L2 |
| Configuración del switch | Aprende múltiples MACs | Solo ve una MAC |
| Caso de uso típico | Legacy, VLANs, rendimiento puro | Cloud, SDN, Kubernetes |

---

## 6.7 None Network: Aislamiento Total

### 6.7.1 Concepto

La red `none` es exactamente lo que su nombre indica: **sin red**. El contenedor no tiene ninguna interfaz de red excepto loopback (`lo`):

```bash
$ docker run --rm --network none alpine ip addr
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
```

No hay `eth0`, no hay bridge, no hay conexión al mundo exterior.

### 6.7.2 Casos de uso

- **Procesos batch o de cómputo**: Un contenedor que procesa datos desde un volumen montado y escribe resultados a otro volumen. No necesita red para nada.
- **Seguridad máxima**: Si un contenedor nunca debe comunicarse con el exterior ni con otros contenedores (ej: generación de claves criptográficas, firma de certificados), `--network none` es la opción más segura.
- **Compilación aislada**: Compilar código potencialmente malicioso en un entorno sin acceso a red.
- **Pruebas de aislamiento**: Verificar que una aplicación funciona correctamente sin conectividad de red.

### 6.7.3 Combinación con otras redes

Puedes arrancar un contenedor con `--network none` y luego conectarle una red específica:

```bash
$ docker run -d --name safe_box --network none alpine sleep infinity
$ docker network connect mi_red safe_box
```

Esto permite un patrón de seguridad donde el contenedor nace sin red y luego se le concede acceso selectivamente.

---

## 6.8 Resolución DNS Interna: Cómo Funciona

### 6.8.1 El DNS embebido de Docker

Docker incluye un servidor DNS interno que corre dentro del daemon (`dockerd`). Cada contenedor que se conecta a una red definida por el usuario recibe automáticamente la dirección de este servidor DNS como su resolver.

La IP del DNS de Docker es **127.0.0.11**. Sí, loopback. ¿Cómo puede un servidor DNS en loopback resolver nombres de otros contenedores? Porque `dockerd` intercepta las peticiones DNS que salen del contenedor y las redirige a su resolver interno mediante reglas de iptables dentro del namespace del contenedor.

```bash
$ docker run --rm --network mi_red alpine cat /etc/resolv.conf
nameserver 127.0.0.11
options ndots:0
```

### 6.8.2 Cómo se configura /etc/resolv.conf

Docker gestiona `/etc/resolv.conf` del contenedor de la siguiente manera:

1. **Por defecto**: Copia el `/etc/resolv.conf` del host, filtrando solo los nameservers que sean localhost (porque no tendrían sentido dentro del namespace del contenedor).
2. **Red definida por el usuario**: Establece `nameserver 127.0.0.11` (el DNS embebido de Docker).

Cuando el DNS embebido recibe una consulta:

- Si el nombre coincide con el nombre de un contenedor en la misma red → devuelve su IP.
- Si el nombre coincide con un alias de red (ver 6.8.6) → devuelve su IP.
- Si el nombre coincide con un servicio Swarm → devuelve la VIP del servicio.
- Si no → reenvía la consulta al DNS externo configurado (normalmente el del host, o los definidos con `--dns`).

### 6.8.3 Configuración DNS personalizada: --dns, --dns-search, --dns-opt

Puedes sobrescribir la configuración DNS:

```bash
$ docker run --rm --network mi_red \
  --dns 8.8.8.8 \
  --dns 1.1.1.1 \
  --dns-search example.com \
  --dns-opt timeout:1 \
  --dns-opt attempts:2 \
  alpine cat /etc/resolv.conf
```

Salida:
```
nameserver 8.8.8.8
nameserver 1.1.1.1
search example.com
options timeout:1 attempts:2
```

- `--dns 8.8.8.8`: Añade este nameserver. Múltiples `--dns` añaden múltiples servidores.
- `--dns-search example.com`: Dominio de búsqueda. Al resolver `app` se intentará `app.example.com` también.
- `--dns-opt`: Opciones de resolución (timeout, attempts, ndots, etc).

Si especificas `--dns`, el DNS embebido de Docker (127.0.0.11) **no** se configura, por lo que perderás la resolución de nombres de contenedores.

**Estrategia**: Puedes combinar el DNS de Docker con servidores externos usando el `--dns` del daemon o definiendo los DNS externos en el propio `/etc/docker/daemon.json`:

```json
{
  "dns": ["8.8.8.8", "1.1.1.1"]
}
```

Con esta configuración, el DNS embebido de Docker reenvía las consultas que no resuelve internamente a los servidores configurados.

### 6.8.4 --hostname: nombre de host del contenedor

Por defecto, Docker asigna al contenedor un hostname igual a su ID (los 12 primeros caracteres). Puedes personalizarlo:

```bash
$ docker run --rm --network mi_red --hostname web-server alpine hostname
web-server
```

Esto cambia el hostname que el contenedor ve internamente, pero **no afecta a la resolución DNS de otros contenedores**. Es decir, otros contenedores no podrán hacer `ping web-server` a menos que el nombre del contenedor sea `web-server` o tenga ese alias de red.

### 6.8.5 Aliases de red: --network-alias

Un contenedor puede tener **múltiples nombres** dentro de una red. Esto se logra con `--network-alias`:

```bash
$ docker run -d --name real_name --network mi_red --network-alias api --network-alias service nginx
```

Otros contenedores en `mi_red` pueden resolverlo por `real_name`, `api`, o `service`:

```bash
$ docker run --rm --network mi_red alpine ping api
PING api (172.18.0.2): 56 data bytes
64 bytes from 172.18.0.2: seq=0 ttl=64 time=0.115 ms

$ docker run --rm --network mi_red alpine ping service
PING service (172.18.0.2): 56 data bytes
64 bytes from 172.18.0.2: seq=0 ttl=64 time=0.089 ms
```

Los aliases de red son la base del descubrimiento de servicios en Docker. Permiten:

- **Balanceo round-robin natural**: Si dos contenedores comparten el mismo alias, el DNS de Docker rota entre sus IPs (ver sección 6.8.6).
- **Renombrar servicios sin cambiar código**: La app se conecta a `api`. Si migras el backend, solo cambias el alias.
- **Azul/verde (blue/green)**: Apuntas el alias `api` a la versión nueva, verificas, y cortas el tráfico.

Puedes añadir y quitar aliases con `docker network connect`:

```bash
$ docker network connect --alias db_write mi_red contenedor
$ docker network disconnect mi_red contenedor
```

### 6.8.6 Resolución DNS y balanceo de carga

Cuando dos contenedores comparten el mismo alias en la misma red, el DNS embebido de Docker rota las respuestas (round-robin), proporcionando balanceo de carga básico a nivel DNS:

```bash
$ docker run -d --name app1 --network mi_red --network-alias app nginx
$ docker run -d --name app2 --network mi_red --network-alias app nginx
$ docker run -d --name app3 --network mi_red --network-alias app nginx

$ docker run --rm --network mi_red alpine nslookup app
Name:      app
Address 1: 172.18.0.2 app1.mi_red
Address 2: 172.18.0.3 app2.mi_red
Address 3: 172.18.0.4 app3.mi_red

$ docker run --rm --network mi_red alpine nslookup app
Name:      app
Address 1: 172.18.0.3 app2.mi_red
Address 2: 172.18.0.4 app3.mi_red
Address 3: 172.18.0.2 app1.mi_red
```

> ¿Ves cómo cambia el orden? Eso es el round-robin del DNS de Docker.

**IMPORTANTE**: Este balanceo es a nivel DNS, no a nivel de conexión. Si un cliente cachea la IP, todas sus peticiones irán al mismo backend. Para balanceo real, usa un reverse proxy o el routing mesh de Swarm.

### 6.8.7 --link (obsoleto): por qué NO usarlo

En las versiones antiguas de Docker (antes de 1.9 y las redes definidas por el usuario), `--link` era la única forma de que los contenedores en el bridge por defecto se resolvieran por nombre:

```bash
# OBSOLETO - NO USAR
$ docker run -d --name db mysql:5.7
$ docker run -d --name app --link db:database myapp
```

`--link` inyectaba variables de entorno y entradas en `/etc/hosts` del contenedor `app`:

```
DB_PORT=tcp://172.17.0.2:3306
DB_PORT_3306_TCP=tcp://172.17.0.2:3306
DATABASE_PORT=tcp://172.17.0.2:3306
...
```

y en `/etc/hosts`:
```
172.17.0.2  database  db
```

**Problemas de --link:**

- **Dependencias ocultas**: Si `db` se reiniciaba y cambiaba de IP, `app` no se enteraba. Había que reiniciar `app` también.
- **Orden de arranque frágil**: `db` tenía que existir antes que `app`, o el comando `docker run` fallaba.
- **Acoplamiento estrecho**: Las variables de entorno filtraban detalles de implementación.
- **Unidireccional**: `app` podía ver a `db`, pero `db` no sabía de `app`.

Con las redes definidas por el usuario y el DNS embebido, `--link` es totalmente innecesario. Docker lo mantiene por compatibilidad hacia atrás, pero **no debe usarse en ningún proyecto nuevo**.

---

## 6.9 Exposición de Puertos: En Detalle

### 6.9.1 -p (publicar un puerto)

La sintaxis completa de `-p` es:

```
-p [ip_del_host:][puerto_host:][puerto_contenedor][/protocolo]
```

#### Caso 1: -p 8080:80

```bash
$ docker run -d -p 8080:80 nginx
```

- Puerto 8080 del host (todas las interfaces: 0.0.0.0:8080) → Puerto 80 del contenedor.
- Protocolo por defecto: TCP.

```
                   +------- iptables DNAT -------+
                   |                             |
   Cliente -----> :8080 (host) ---------------> :80 (contenedor)
                   |
           docker-proxy (opcional)
```

#### Caso 2: -p 127.0.0.1:8080:80

```bash
$ docker run -d -p 127.0.0.1:8080:80 nginx
```

- Solo localhost del host. Conexiones desde la red externa no pueden alcanzar el puerto 8080.
- Útil para servicios que solo deben ser accesibles desde el host (ej: panel de administración, base de datos).

```bash
$ curl localhost:8080     # funciona
$ curl 192.168.1.100:8080 # no funciona
```

#### Caso 3: -p 80

```bash
$ docker run -d -p 80 nginx
```

- Publica el puerto 80 del contenedor en un puerto aleatorio alto del host (rango efímero, típicamente 32768-65535).

```bash
$ docker port romantic_einstein
80/tcp -> 0.0.0.0:32768
```

#### Caso 4: -p 8080:80/udp

```bash
$ docker run -d -p 8080:80/udp -p 8080:80/tcp nginx
```

- Especificas el protocolo explícitamente. Si no pones nada, es `tcp`.
- Para publicar tanto TCP como UDP en el mismo puerto, necesitas dos `-p`.

### 6.9.2 -P (publish all)

Publica automáticamente todos los puertos declarados con `EXPOSE` en el Dockerfile a puertos aleatorios del host:

```bash
$ docker run -d -P nginx
$ docker port hungry_bell
80/tcp -> 0.0.0.0:32769
```

Si la imagen tiene `EXPOSE 80 443`, ambos se publican a puertos aleatorios:

```bash
$ docker run -d -P myimage
$ docker port mystifying_pasteur
80/tcp -> 0.0.0.0:32770
443/tcp -> 0.0.0.0:32771
```

`EXPOSE` es puramente documental (no abre puertos por sí solo), pero `-P` lo utiliza para saber qué puertos publicar.

### 6.9.3 Rangos de puertos

Puedes publicar un rango de puertos:

```bash
$ docker run -d -p 8080-8090:80 nginx
```

Esto publica los puertos del host 8080, 8081, 8082... 8090, todos apuntando al puerto 80 del contenedor.

También puedes mapear rangos:

```bash
$ docker run -d -p 8000-8010:8000-8010 mi_app
```

El puerto 8000 del host → 8000 del contenedor, 8001 → 8001, etc. Ambos rangos deben tener la misma longitud.

### 6.9.4 Cómo funciona iptables (el viaje de un paquete)

Veamos el viaje completo de una petición HTTP externa a un contenedor:

```
         CLIENTE (192.168.1.50)
                |
                | Paquete: SRC=192.168.1.50:54321, DST=192.168.1.100:8080
                v
    +-----------------------+
    |         eth0          |  Host (192.168.1.100)
    |    (interfaz física)  |
    +-----------+-----------+
                |
                v
    +-----------------------+
    |    PREROUTING (nat)   |  Cadena de iptables
    |    Regla DNAT:        |
    |    dst:8080 ->        |
    |    dst:172.17.0.2:80  |
    +-----------+-----------+
                |
                | Paquete: SRC=192.168.1.50:54321, DST=172.17.0.2:80
                v
    +-----------------------+
    |      FORWARD          |  Reenvío de paquetes (debe estar habilitado)
    |    (filter)           |  net.ipv4.ip_forward = 1
    +-----------+-----------+
                |
                v
    +-----------------------+
    |     docker0 bridge    |  Switch virtual L2
    |                       |
    |  Conmuta al puerto    |
    |  del veth del cont.   |
    +-----------+-----------+
                |
                v
    +-----------------------+
    |  Contenedor Nginx     |  IP: 172.17.0.2
    |  eth0 (veth)          |  Recibe en puerto 80
    |                       |
    |  Procesa la petición  |
    |  y responde...        |
    +-----------------------+
```

**Respuesta (salida):**

```
    +-----------------------+
    |  Contenedor Nginx     |
    |  Respuesta:           |
    |  SRC=172.17.0.2:80    |
    |  DST=192.168.1.50:... |
    +-----------+-----------+
                |
                v
    +-----------------------+
    |     docker0 bridge    |
    +-----------+-----------+
                |
                v
    +-----------------------+
    |   POSTROUTING (nat)   |  MASQUERADE (SNAT)
    |   SRC: 172.17.0.2 ->  |  Cambia IP origen
    |   SRC: 192.168.1.100  |  a la IP del host
    +-----------+-----------+
                |
                v
    +-----------------------+
    |         eth0          |  Sale al cliente
    +-----------------------+
```

**Sin MASQUERADE**, el cliente recibiría una respuesta desde `172.17.0.2` (una IP de Docker interna) y la descartaría porque no esperaba esa IP.

Para tráfico de salida iniciado por el contenedor (por ejemplo, `curl google.com` desde dentro), aplica la misma regla MASQUERADE en POSTROUTING: la IP origen privada se reemplaza por la IP del host.

### 6.9.5 Listando las reglas de iptables de Docker

```bash
# Ver cadenas de Docker en tabla nat
$ sudo iptables -t nat -L -n -v

# DOCKER chain: reglas DNAT para puertos publicados
$ sudo iptables -t nat -L DOCKER -n -v

# POSTROUTING chain: reglas MASQUERADE
$ sudo iptables -t nat -L POSTROUTING -n -v

# FORWARD chain: filtrado entre interfaces
$ sudo iptables -L FORWARD -n -v

# DOCKER-ISOLATION chain: aislamiento entre bridges
$ sudo iptables -L DOCKER-ISOLATION -n -v
```

La cadena `DOCKER-ISOLATION` es interesante: Docker añade reglas para que contenedores en diferentes redes bridge no puedan comunicarse directamente. El tráfico entre bridges se bloquea (DROP) y solo se permite si los contenedores comparten al menos una red.

---

## 6.10 Redes en Docker Compose

### 6.10.1 La red por defecto de Compose

Cuando ejecutas `docker compose up`, Compose crea automáticamente una red bridge para tu proyecto:

```yaml
# docker-compose.yml
version: '3.8'
services:
  web:
    image: nginx
  db:
    image: mysql:8
```

```bash
$ docker compose up -d
$ docker network ls
NETWORK ID          NAME                    DRIVER              SCOPE
abc123def456        proyecto_default        bridge              local
```

El nombre de la red sigue el patrón `<directorio_del_proyecto>_default`. Todos los servicios definidos en el compose file se conectan automáticamente a esta red.

**Resolución DNS en Compose**: Cada servicio puede resolver a cualquier otro por su nombre de servicio (`web`, `db`). No necesitas `links` ni `--network-alias` para lo básico.

### 6.10.2 Redes definidas explícitamente (top-level networks)

Puedes definir redes personalizadas a nivel `networks:` (top-level) y asignar servicios a ellas:

```yaml
version: '3.8'
services:
  web:
    image: nginx
    networks:
      - frontend

  api:
    image: mi_api:latest
    networks:
      - frontend
      - backend

  db:
    image: mysql:8
    networks:
      - backend

networks:
  frontend:
    driver: bridge
  backend:
    driver: bridge
    internal: true
```

```
+----------------------------------------------------------+
|                    PROYECTO COMPOSE                       |
|                                                          |
|   +------------+      +------------+      +------------+ |
|   |    web     |      |    api     |      |     db     | |
|   |            |      |            |      |            | |
|   | frontend --+------+-- frontend |      |            | |
|   |            |      |            |      |            | |
|   |            |      | backend ---+------+-- backend  | |
|   +------------+      +------------+      +------------+ |
|        |                    |                    |        |
|        v                    v                    v        |
|   [frontend]           [frontend]           [backend]     |
|   (bridge)             [backend]            (internal)    |
|                        (bridge)                          |
+----------------------------------------------------------+
```

En esta arquitectura:
- `web` solo ve a `api` (está en `frontend`).
- `api` ve tanto a `web` (en `frontend`) como a `db` (en `backend`).
- `db` solo ve a `api` (está en `backend`).
- `db` está en una red `internal`: no tiene acceso a internet.

### 6.10.3 Configuración de red por servicio

```yaml
version: '3.8'
services:
  api:
    image: mi_api:latest
    networks:
      frontend:
        aliases:
          - api_v1
          - api_legacy
        ipv4_address: 172.20.0.100
      backend:
        priority: 1000

networks:
  frontend:
    driver: bridge
    ipam:
      config:
        - subnet: 172.20.0.0/16
          gateway: 172.20.0.1
  backend:
    driver: bridge
    internal: true
```

Opciones disponibles por servicio en `networks:`:
- `aliases`: Nombres alternativos para el servicio en esa red.
- `ipv4_address`: IP fija (requiere que la red tenga configurado `ipam` con la subred).
- `ipv6_address`: IP fija IPv6.
- `priority`: Orden en que se conectan las redes (valores más altos = primera red conectada).

### 6.10.4 Redes externas (external: true)

Si una red ya existe fuera de Compose (creada con `docker network create`), puedes referenciarla como externa:

```yaml
version: '3.8'
services:
  web:
    image: nginx
    networks:
      - red_compartida

networks:
  red_compartida:
    external: true
```

Compose no intentará crear la red. Si no existe, fallará:

```bash
$ docker compose up
Network red_compartida declared as external, but could not be found.
```

Esto permite compartir redes entre múltiples proyectos Compose o entre contenedores gestionados manualmente y stacks de Compose.

### 6.10.5 Redes compartidas entre múltiples Compose files

**Escenario**: Proyecto A (frontend) y Proyecto B (backend API) necesitan comunicarse pero mantenerse independientes.

```bash
# Primero, crear la red compartida
$ docker network create red_shared
```

**Proyecto A (frontend/docker-compose.yml):**
```yaml
version: '3.8'
services:
  nginx:
    image: nginx
    networks:
      - red_shared

networks:
  red_shared:
    external: true
```

**Proyecto B (backend/docker-compose.yml):**
```yaml
version: '3.8'
services:
  api:
    image: mi_api
    networks:
      - red_shared

networks:
  red_shared:
    external: true
```

Ambos stacks pueden resolverse por nombre de servicio (el nombre del servicio en cada compose file no colisiona con el del otro). `nginx` puede hacer ping a `api` y viceversa.

### 6.10.6 network_mode

`network_mode` permite modos de red especiales a nivel de servicio:

#### network_mode: "host"

```yaml
version: '3.8'
services:
  proxy:
    image: haproxy:latest
    network_mode: host
    # Los puertos se toman directamente del host. No se necesita ports:
```

#### network_mode: "none"

```yaml
version: '3.8'
services:
  batch_processor:
    image: mi_processor
    network_mode: none
```

#### network_mode: "service:<nombre>"

Este es un modo muy potente: el servicio comparte **toda la pila de red** de otro servicio:

```yaml
version: '3.8'
services:
  app:
    image: nginx
    ports:
      - "8080:80"

  sidecar:
    image: alpine
    network_mode: "service:app"
    command: |
      sh -c "
        apk add --no-cache curl &&
        while true; do
          curl -s http://localhost:80 > /dev/null &&
          echo 'OK' || echo 'FAIL';
          sleep 5;
        done
      "
```

El contenedor `sidecar` comparte el namespace de red de `app`. Significa que:
- `sidecar` ve `localhost:80` como el puerto 80 de `app` (no necesita saber la IP de `app`).
- Ambos comparten la misma interfaz de red y la misma IP.
- Los puertos publicados de `app` son accesibles desde `sidecar` vía loopback.

Este patrón (sidecar) es fundamental en arquitecturas de microservicios modernas (Istio/Envoy, proxies, health checks locales, log shippers como Fluentd).

#### network_mode: "container:<id>"

```yaml
services:
  monitor:
    image: nicolaka/netshoot
    network_mode: "container:nombre_contenedor_existente"
```

Similar a `service:<nombre>`, pero apunta a un contenedor existente (no gestionado por este compose file).

### 6.10.7 Puntos clave de redes en Compose

- Los servicios sin `networks:` especificado se conectan a la red `default` automática.
- Si defines **alguna** red explícita para un servicio, la red `default` **no** se conecta. Si quieres que también esté en `default`, debes especificarlo explícitamente: `networks: - default - mi_red`.
- Los `aliases` funcionan igual que `--network-alias`: múltiples nombres DNS para el mismo contenedor.
- DNS resuelve nombres de servicio dentro del mismo compose file automáticamente.

### 6.10.8 Patrones avanzados de redes en Compose

#### Patrón: API Gateway con red privada

```yaml
version: '3.8'
services:
  gateway:
    image: nginx:alpine
    ports:
      - "443:443"
    networks:
      - public
      - private
    volumes:
      - ./gateway.conf:/etc/nginx/nginx.conf:ro

  auth:
    image: mi_auth
    networks:
      - private

  user_service:
    image: mi_users
    networks:
      - private

  order_service:
    image: mi_orders
    networks:
      - private

networks:
  public:
    driver: bridge
  private:
    driver: bridge
    internal: true
```

Todos los microservicios están en `private` (sin acceso a internet, sin puertos expuestos). Solo el `gateway` tiene pie en `public` (con puertos expuestos) y `private`. El gateway enruta peticiones externas autenticadas hacia los servicios internos. Los servicios internos no pueden salir a internet ni ser accedidos directamente.

#### Patrón: Bases de datos replicadas con red dedicada

```yaml
version: '3.8'
services:
  app:
    image: mi_app
    networks:
      - app_net
      - db_net

  db_primary:
    image: mysql:8
    networks:
      - db_net
    environment:
      MYSQL_ROOT_PASSWORD: pass
    volumes:
      - db_primary:/var/lib/mysql

  db_replica:
    image: mysql:8
    networks:
      - db_net
    environment:
      MYSQL_ROOT_PASSWORD: pass
    volumes:
      - db_replica:/var/lib/mysql

networks:
  app_net:
    driver: bridge
  db_net:
    driver: bridge
    internal: true

volumes:
  db_primary:
  db_replica:
```

La red `db_net` es `internal` y solo existe para comunicación entre `app`, `db_primary` y `db_replica`. La app se conecta a `db_primary` para escrituras y a `db_replica` para lecturas. Ninguna base de datos está expuesta al exterior. Si un atacante compromete la app, no puede usar la base de datos como pivot para salir a internet porque `db_net` es `internal`.

#### Patrón: Microservicios efímeros con alias de red

```yaml
version: '3.8'
services:
  worker:
    image: mi_worker
    networks:
      queue_net:
        aliases:
          - processor
          - job_runner
    deploy:
      replicas: 10

  redis:
    image: redis:7-alpine
    networks:
      queue_net:

networks:
  queue_net:
    driver: bridge
```

Las 10 réplicas de `worker` comparten los alias `processor` y `job_runner`. Cualquier servicio que resuelva `processor` obtiene una IP diferente (round-robin), distribuyendo el trabajo sin orquestador externo.

### 6.10.9 Orden de conexión a redes: PRIORITY

Cuando un servicio Compose se conecta a múltiples redes, el orden importa. La primera red en la lista se convierte en la red por defecto (la que tiene la ruta `default`). Si necesitas cambiar esto:

```yaml
services:
  api:
    networks:
      frontend:         # 1ra red listada = ruta por defecto
        priority: 1000
      backend:
        priority: 500
```

El parámetro `priority` (disponible desde Compose 3.8+ con Docker Engine 20.10+) establece el orden en que se conectan las redes. Valores más altos = primero. Esto permite controlar cuál de las redes asume la ruta por defecto en el contenedor, independientemente de la posición en el YAML.

---

## 6.11 Troubleshooting de Red: Herramientas Prácticas

### 6.11.1 docker network inspect

El comando más útil para entender qué está pasando:

```bash
$ docker network inspect mi_red
```

Te da:
- Subred y gateway.
- Lista de contenedores conectados con sus IPs, MACs y nombres.
- Opciones del driver.

Para formatear la salida (solo IPs y nombres):

```bash
$ docker network inspect mi_red \
  --format '{{range .Containers}}{{.Name}}: {{.IPv4Address}}{{"\n"}}{{end}}'
app1: 172.18.0.2/16
app2: 172.18.0.3/16
db: 172.18.0.4/16
```

### 6.11.2 Ver interfaces de red del contenedor

```bash
$ docker exec app1 ip addr
$ docker exec app1 ip route
```

Alternativa si el contenedor no tiene `ip`:

```bash
$ docker exec app1 cat /etc/hosts
$ docker exec app1 cat /etc/resolv.conf
```

### 6.11.3 Probar conectividad

```bash
# Ping entre contenedores en misma red
$ docker exec app1 ping app2

# Ping al exterior
$ docker exec app1 ping 8.8.8.8

# HTTP
$ docker exec app1 wget -O- http://app2:8080
$ docker exec app1 curl http://app2:8080
```

### 6.11.4 Resolución DNS

```bash
# nslookup (si el contenedor tiene busybox o bind-tools)
$ docker exec app1 nslookup app2

# Alternativa: dig
$ docker exec app1 dig app2

# Si no hay herramientas DNS, usar getent
$ docker exec app1 getent hosts app2
172.18.0.3    app2
```

### 6.11.5 nicolaka/netshoot: la navaja suiza de red

`netshoot` es una imagen Docker diseñada específicamente para troubleshooting de red. Incluye: `iperf`, `tcpdump`, `nmap`, `curl`, `dig`, `netstat`, `ss`, `iftop`, `drill`, `tcptraceroute`, `mtr` y docenas de herramientas más.

**Modo 1: Contenedor efímero en la red a investigar**

```bash
$ docker run --rm -it --network mi_red nicolaka/netshoot
```

Dentro, tienes todas las herramientas:

```bash
bash-5.0# ping app1
bash-5.0# dig app2
bash-5.0# nmap -p 80 app1
bash-5.0# curl http://app1:80
bash-5.0# tcpdump -i eth0 -n port 80
```

**Modo 2: Usar el namespace de red de otro contenedor**

```bash
$ docker run --rm -it --network container:app1 nicolaka/netshoot
```

Esto es mágico: `netshoot` comparte la pila de red de `app1`. Puedes hacer `tcpdump` del tráfico que ve `app1` sin instalar nada en `app1`:

```bash
bash-5.0# tcpdump -i eth0 -n 'port 80 or port 443'
bash-5.0# ss -tlnp
bash-5.0# ip addr
bash-5.0# ip route
```

Esta técnica es invaluable para depurar problemas en producción donde no puedes modificar el contenedor original.

### 6.11.6 nsenter: entrar al namespace de red desde el host

Si no puedes o no quieres usar `netshoot`, puedes usar `nsenter` desde el host para inspeccionar el namespace de red de un contenedor:

```bash
# Obtener el PID del contenedor
$ CONTAINER_PID=$(docker inspect -f '{{.State.Pid}}' app1)

# Entrar al namespace de red del contenedor
$ sudo nsenter -t $CONTAINER_PID -n ip addr
$ sudo nsenter -t $CONTAINER_PID -n ping 8.8.8.8
$ sudo nsenter -t $CONTAINER_PID -n ss -tlnp
```

- `-t $PID`: El proceso objetivo (el contenedor).
- `-n`: Entrar al network namespace.
- Comandos posteriores se ejecutan en el namespace de red del contenedor.

Para un "shell completo" en todos los namespaces del contenedor:

```bash
$ sudo nsenter -t $CONTAINER_PID -m -u -i -n -p bash
```

### 6.11.7 Capturar tráfico con tcpdump

Dentro de un contenedor (requiere tcpdump instalado o usar netshoot):

```bash
# Capturar todo el tráfico en eth0 del contenedor
$ docker run --rm --network container:app1 nicolaka/netshoot \
  tcpdump -i eth0 -n -w /tmp/capture.pcap

# Filtrar por puerto
$ docker run --rm --network container:app1 nicolaka/netshoot \
  tcpdump -i eth0 -n 'port 3306'

# Capturar tráfico DNS
$ docker run --rm --network container:app1 nicolaka/netshoot \
  tcpdump -i eth0 -n 'port 53'
```

### 6.11.8 Problemas comunes y soluciones

#### Problema 1: "El contenedor no puede acceder a internet"

**Síntomas**:
```bash
$ docker exec app1 ping 8.8.8.8
ping: sendto: Network is unreachable
```

**Diagnóstico**:
```bash
# Verificar que IP forwarding está habilitado en el host
$ sysctl net.ipv4.ip_forward
net.ipv4.ip_forward = 0   # <-- PROBLEMA

# Verificar reglas MASQUERADE
$ sudo iptables -t nat -L POSTROUTING -n -v
# Debe haber una regla MASQUERADE para la subred del bridge
```

**Solución**:
```bash
$ sudo sysctl -w net.ipv4.ip_forward=1

# Persistente en /etc/sysctl.conf o /etc/sysctl.d/
$ echo 'net.ipv4.ip_forward = 1' | sudo tee /etc/sysctl.d/99-docker.conf
$ sudo sysctl --system

# Reiniciar Docker
$ sudo systemctl restart docker
```

#### Problema 2: "No puedo conectar al puerto publicado desde otra máquina"

**Síntomas**: `curl localhost:8080` funciona en el host, pero `curl 192.168.1.100:8080` desde otra máquina no.

**Diagnóstico**:
```bash
# Verificar que el puerto está escuchando en 0.0.0.0
$ ss -tlnp | grep 8080
LISTEN  0  128  0.0.0.0:8080  0.0.0.0:*   users:(("docker-proxy",...))

# Si muestra 127.0.0.1:8080, está vinculado solo a localhost
```

**Solución**:
- Ejecutar con `-p 0.0.0.0:8080:80` para forzar todas las interfaces.
- Revisar firewall: `sudo ufw status`, `sudo iptables -L INPUT -n -v`.
- En macOS (Docker Desktop): el tráfico externo no llega directamente al host. Docker Desktop usa una VM intermedia. Mapea puertos en la configuración o usa `localhost` desde el host macOS.

#### Problema 3: "Dos contenedores no se ven en el bridge por defecto"

**Síntomas**: `ping` por IP funciona, pero por nombre no.

**Causa**: El bridge por defecto no tiene DNS automático. Usa redes definidas por el usuario.

#### Problema 4: "Contenedor pierde conectividad después de un tiempo"

**Diagnóstico**:
```bash
# Verificar logs de Docker
$ sudo journalctl -u docker -f

# Verificar que la red no ha cambiado
$ docker network inspect mi_red

# Verificar reglas de iptables (firewall puede estar eliminándolas)
$ sudo iptables -L -n -v | grep DOCKER
```

**Solución**: Muchos servicios de firewall (firewalld, ufw con configuraciones agresivas) interfieren con iptables de Docker. Configura el firewall para respetar las cadenas de Docker o usa `--iptables=false` en el daemon (y gestiona manualmente).

#### Problema 5: "Conflicto de subred de Docker con VPN/red corporativa"

**Síntomas**: No puedes acceder a recursos de la red `10.0.0.0/8` desde el host porque Docker usa `172.17.0.0/16` que no debería colisionar... o al revés, Docker usa `10.0.X.X` y colisiona.

**Solución**: Configurar el bridge por defecto en `/etc/docker/daemon.json`:

```json
{
  "bip": "192.168.100.1/24",
  "default-address-pools": [
    {
      "base": "192.168.200.0/24",
      "size": 24
    }
  ]
}
```

- `bip` (Bridge IP): IP y subred del bridge `docker0`.
- `default-address-pools`: Piscinas de subredes para redes definidas por el usuario.

Reinicia Docker tras el cambio:
```bash
$ sudo systemctl restart docker
```

#### Problema 6: "DNS no resuelve nombres de contenedor"

**Síntomas**:
```bash
$ docker exec app1 ping app2
ping: bad address 'app2'
```

**Diagnóstico**:
```bash
# Verificar que el contenedor está en una red de usuario
$ docker inspect app1 | grep -A 10 Networks
# Debe estar en una red bridge definida por el usuario, no en "bridge" por defecto

# Verificar /etc/resolv.conf del contenedor
$ docker exec app1 cat /etc/resolv.conf
# Debe tener "nameserver 127.0.0.11"
```

**Solución**: Asegurar que ambos contenedores están en la misma red **definida por el usuario**. Si uno está en `bridge` (por defecto) y otro en `mi_red`, no se verán por DNS.

#### Problema 7: "Overlay no comunica entre nodos Swarm"

**Síntomas**: Contenedores en el mismo overlay en diferentes nodos no se alcanzan.

**Diagnóstico**:
```bash
# Verificar conectividad UDP en puerto 4789 (VXLAN)
$ nc -zu 192.168.1.20 4789  # desde el manager hacia el worker

# Verificar que los puertos de Swarm están abiertos
$ nc -zu 192.168.1.20 7946  # Serf/gossip

# En el manager, ver nodos activos
$ docker node ls
ID      HOSTNAME   STATUS    AVAILABILITY   MANAGER STATUS
abc *   node1      Ready     Active         Leader
def     node2      Down      Active         # <-- PROBLEMA
```

**Solución**:
- Abrir puertos 2377/tcp, 7946/tcp+udp, 4789/udp en todos los nodos.
- Verificar que los nodos pueden resolver los hostnames entre sí (DNS o `/etc/hosts`).
- Comprobar que `--advertise-addr` del manager es accesible desde los workers.
- Revisar MTU: si los paquetes VXLAN + headers superan la MTU de la red física, se fragmentan o descartan. Reducir MTU de la red Docker (`-o com.docker.network.driver.mtu=1400`).

#### Problema 8: "Conflictos de puertos en redes macvlan con múltiples contenedores"

**Síntomas**: Los contenedores macvlan obtienen IPs pero no se comunican entre sí.

**Diagnóstico**:
```bash
# Verificar que el parent existe y está en modo bridge
$ sudo ip -d link show eth0
# Debe mostrar "macvlan mode bridge"

$ cat /sys/class/net/eth0/operstate
up
```

**Solución**: 
- Asegurar que la interfaz física tiene modo promiscuo habilitado: `sudo ip link set eth0 promisc on`.
- En entornos virtualizados (VMware, VirtualBox), habilitar "Promiscuous mode" en la configuración del adaptador de red de la VM.
- Para cloud (AWS, GCP, Azure): macvlan generalmente no funciona sin soporte del hipervisor. Usa ipvlan en modo L3 como alternativa.

#### Problema 9: "La app se conecta a MySQL pero las consultas van lentas"

**Síntomas**: Conexión correcta pero latencia alta inesperada (>5ms para consulta simple).

**Diagnóstico**:
```bash
# Verificar latencia de red entre contenedores
$ docker exec app1 ping -c 10 db
--- db ping statistics ---
10 packets transmitted, 10 received, 0% loss, time 45ms
rtt min/avg/max/mdev = 4.201/4.510/4.901/0.212 ms

# Verificar si el tráfico sale por la interfaz correcta
$ docker exec app1 ip route get 172.21.0.2
172.21.0.2 dev eth1 src 172.21.0.4 uid 0
```

¿La ruta pasa por `eth1` (interfaz de backend, directa) o por `eth0` (interfaz de frontend, que requiere NAT)? Si la app tiene múltiples redes y la ruta por defecto apunta a la red con acceso a internet, la conexión a la base de datos podría estar saliendo por la ruta equivocada y pasando por iptables innecesariamente.

**Solución**: Ajustar el orden de las redes en Compose con `priority` para que la red interna (sin NAT) sea la primera y por tanto la ruta por defecto.

---

### 6.11.9 Plugins de red de terceros

Docker permite extender las capacidades de red mediante plugins. Mientras que los drivers nativos cubren la mayoría de casos, los plugins de terceros ofrecen funcionalidades avanzadas para Kubernetes, SDN, políticas de red y entornos cloud.

#### Calico

**Calico** es un plugin de red L3 puro (sin overlay). En lugar de encapsular paquetes, configura rutas BGP en cada host para que los paquetes lleguen directamente a los pods/contenedores destino. Es la opción predominante en Kubernetes.

```
        +-------+       +--------+       +-------+
        | Host1 |       | Router |       | Host2 |
        |       |       |  BGP   |       |       |
        | [Calico] <---|  route |-----> [Calico] |
        |       |  BGP  | reflect|  BGP  |       |
        +---+---+       +--------+       +---+---+
            |                                 |
    Contenedor A                      Contenedor B
    10.0.1.2                          10.0.2.3
```

**Ventajas:**
- Sin encapsulamiento (sin overhead VXLAN), rendimiento cercano al nativo.
- NetworkPolicy: reglas de firewall entre pods (Kubernetes y Calico).
- Soporte para BGP, enrutamiento directo sin NAT.

**Desventaja**: Requiere que la red subyacente soporte BGP o que IPIP/VXLAN se use como fallback.

**Instalación en Docker:**
```bash
$ docker plugin install calico/calico
$ docker network create --driver calico --ipam-driver calico-ipam calico_net
```

#### Weave Net

**Weave** crea una red overlay L2 con su propio protocolo de enrutamiento distribuido. A diferencia de VXLAN, Weave usa su propio encabezado Weave (más ligero que VXLAN) y encriptación opcional.

```bash
$ sudo curl -L git.io/weave -o /usr/local/bin/weave
$ sudo chmod +x /usr/local/bin/weave
$ weave launch
$ weave launch-proxy
$ docker run --rm alpine ping contenedor_weave
```

**Ventajas:**
- Fácil de instalar (un binario, sin dependencias).
- Encriptación NaCl integrada.
- Descubrimiento automático de nodos.
- Funciona sin Swarm, sin KV store.

#### Flannel

**Flannel** fue el primer plugin de red diseñado para Kubernetes. Crea una red overlay simple asignando una subred a cada host y encapsulando tráfico entre hosts con VXLAN o UDP.

```bash
# Configurar etcd con la configuración de flannel
$ etcdctl set /coreos.com/network/config '{"Network":"10.5.0.0/16"}'
$ flanneld &
```

Docker puede usar la red de flannel configurando el bridge por defecto en `/etc/docker/daemon.json`:
```json
{
  "bip": "10.5.1.1/24",
  "mtu": 1472
}
```

#### Cilium

**Cilium** es un plugin de red moderno basado en eBPF (un subsistema del kernel de Linux para programación segura). Inspecciona tráfico L7 (HTTP, Kafka, gRPC) y aplica políticas de seguridad a nivel de API.

**Ventajas:**
- Visibilidad y filtrado L7 (por ejemplo, "el servicio A solo puede hacer GET a /users, no POST").
- eBPF es extremadamente eficiente (sin iptables, sin proxies).
- Integración con Hubble para observabilidad.

**Desventaja**: Requiere kernel Linux moderno (>=4.9 para funcionalidades básicas, >=5.x para avanzadas).

### 6.11.10 Comparativa de soluciones de red

| Solución | Tipo | Encriptación | NetworkPolicy | Cloud | Rendimiento |
|----------|------|-------------|---------------|-------|-------------|
| **Bridge Docker** | L2 bridge | No | ICC flag | N/A | Medio |
| **Overlay Docker** | VXLAN | IPSec opcional | No | Sí | Medio-bajo |
| **Macvlan** | L2 directa | No | No | Limitado | Máximo |
| **Calico** | L3/BGP | WireGuard | Sí | Sí | Alto |
| **Weave** | Overlay propio | NaCl | Sí | Sí | Medio |
| **Flannel** | VXLAN/UDP | No | No | Sí | Medio |
| **Cilium** | eBPF | IPsec/WG | Sí (L7) | Sí | Alto |

---

## 6.12 Laboratorio: Arquitectura de Microservicios con Redes

### 6.12.1 Objetivo

Construir una arquitectura de microservicios completa usando solo redes de Docker. Sin Kubernetes, sin Swarm, sin balanceadores externos. Solo Docker, Compose y redes bien diseñadas.

### 6.12.2 Arquitectura

```
+----------------------------------------------------------+
|              ARQUITECTURA DE MICROSERVICIOS               |
|                                                          |
|   INTERNET                                               |
|      |                                                   |
|      v                                                   |
|  +--------+      RED: frontend (bridge)                  |
|  | Nginx  |                                              |
|  | (LB)   |----+----+----+----+----+                    |
|  +--------+    |    |    |    |    |                     |
|                v    v    v    v    v                     |
|            +----++----++----++----++----+               |
|            |app1||app2||app3||app4||app5|               |
|            +----++----++----++----++----+               |
|              |     |     |     |     |                   |
|              +-----+-----+-----+-----+                   |
|                      |     |                             |
|                      v     v                             |
|              RED: backend (bridge, internal)             |
|                      |     |                             |
|                  +---+--+ ++---+--+                      |
|                  | MySQL | | Redis |                     |
|                  +-------+ +------+                      |
|                                                          |
+----------------------------------------------------------+
```

**Diseño de redes:**
- `frontend`: Red donde residen el balanceador (Nginx) y las réplicas de la aplicación. Tiene acceso a internet.
- `backend`: Red interna sin acceso a internet (`internal: true`). Alberga MySQL y Redis. Las aplicaciones se conectan aquí para acceder a datos.

**Flujo de tráfico:**
1. Usuario → Nginx (puerto 80, red `frontend`).
2. Nginx → alguna de las 5 réplicas de `app` (por DNS round-robin en `frontend`).
3. App → MySQL o Redis (por DNS en `backend`).

MySQL y Redis **nunca** tienen puertos expuestos al exterior. Solo las réplicas de `app` (que están en ambas redes) pueden alcanzarlos.

### 6.12.3 Archivos del laboratorio

#### Estructura de directorios

```
laboratorio-redes/
  docker-compose.yml
  nginx/
    nginx.conf
  app/
    Dockerfile
    app.py
    requirements.txt
  db/
    init.sql
```

#### db/init.sql

```sql
CREATE DATABASE IF NOT EXISTS app_db;
USE app_db;

CREATE TABLE IF NOT EXISTS counters (
  id INT PRIMARY KEY AUTO_INCREMENT,
  service_name VARCHAR(50),
  hits INT DEFAULT 0,
  last_updated TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
);

INSERT INTO counters (service_name, hits) VALUES ('app1', 0), ('app2', 0), ('app3', 0), ('app4', 0), ('app5', 0)
ON DUPLICATE KEY UPDATE service_name=service_name;
```

#### app/app.py

```python
import os
import socket
import redis
import pymysql
from flask import Flask, jsonify, request

app = Flask(__name__)

SERVICE_NAME = socket.gethostname()
DB_HOST = os.environ.get('DB_HOST', 'db')
DB_USER = os.environ.get('DB_USER', 'app_user')
DB_PASSWORD = os.environ.get('DB_PASSWORD', 'app_pass')
DB_NAME = os.environ.get('DB_NAME', 'app_db')
REDIS_HOST = os.environ.get('REDIS_HOST', 'cache')

def get_db_connection():
    return pymysql.connect(
        host=DB_HOST,
        user=DB_USER,
        password=DB_PASSWORD,
        database=DB_NAME,
        connect_timeout=3
    )

@app.route('/')
def index():
    try:
        conn = get_db_connection()
        cursor = conn.cursor()
        cursor.execute(
            "UPDATE counters SET hits = hits + 1 WHERE service_name = %s",
            (SERVICE_NAME,)
        )
        conn.commit()
        cursor.execute("SELECT service_name, hits FROM counters")
        rows = cursor.fetchall()
        conn.close()

        return jsonify({
            'service': SERVICE_NAME,
            'hostname': socket.gethostname(),
            'remote_addr': request.remote_addr,
            'counters': {row[0]: row[1] for row in rows}
        })
    except Exception as e:
        return jsonify({
            'service': SERVICE_NAME,
            'error': str(e)
        }), 500

@app.route('/health')
def health():
    checks = {
        'database': False,
        'redis': False
    }
    try:
        conn = get_db_connection()
        conn.close()
        checks['database'] = True
    except:
        pass

    try:
        r = redis.Redis(host=REDIS_HOST, port=6379, socket_connect_timeout=2)
        r.ping()
        checks['redis'] = True
    except:
        pass

    return jsonify(checks)

@app.route('/cache/<key>')
def get_cache(key):
    try:
        r = redis.Redis(host=REDIS_HOST, port=6379, socket_connect_timeout=2)
        value = r.get(key)
        return jsonify({key: value.decode() if value else None})
    except Exception as e:
        return jsonify({'error': str(e)}), 500

@app.route('/cache/<key>', methods=['POST'])
def set_cache(key):
    try:
        r = redis.Redis(host=REDIS_HOST, port=6379, socket_connect_timeout=2)
        value = request.json.get('value', '')
        r.set(key, value)
        return jsonify({'status': 'ok', key: value})
    except Exception as e:
        return jsonify({'error': str(e)}), 500

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)
```

#### app/requirements.txt

```
flask
pymysql
redis
```

#### app/Dockerfile

```dockerfile
FROM python:3.11-alpine
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY app.py .
EXPOSE 5000
CMD ["python", "app.py"]
```

#### nginx/nginx.conf

```nginx
events {
    worker_connections 1024;
}

http {
    upstream app_backend {
        server app:5000;
    }

    server {
        listen 80;

        location / {
            proxy_pass http://app_backend;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
        }

        location /health {
            proxy_pass http://app_backend/health;
            proxy_set_header Host $host;
        }
    }
}
```

Observa `upstream app_backend { server app:5000; }`. Esto es clave: Nginx se conecta a `app:5000`. Gracias al DNS de Docker, `app` se resuelve a una de las 5 réplicas (round-robin), proporcionando balanceo de carga **sin configurar IPs ni puertos de host**. Solo funciona porque Nginx y las apps están en la misma red `frontend`.

#### docker-compose.yml

```yaml
version: '3.8'

services:
  lb:
    image: nginx:alpine
    ports:
      - "8080:80"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro
    networks:
      - frontend
    depends_on:
      - app

  app:
    build: ./app
    # Sin ports: las apps no son accesibles directamente
    environment:
      - DB_HOST=db
      - DB_USER=app_user
      - DB_PASSWORD=app_pass
      - DB_NAME=app_db
      - REDIS_HOST=cache
    networks:
      - frontend
      - backend
    deploy:
      replicas: 5

  db:
    image: mysql:8.0
    environment:
      MYSQL_ROOT_PASSWORD: rootpass
      MYSQL_DATABASE: app_db
      MYSQL_USER: app_user
      MYSQL_PASSWORD: app_pass
    volumes:
      - db_data:/var/lib/mysql
      - ./db/init.sql:/docker-entrypoint-initdb.d/init.sql:ro
    networks:
      - backend

  cache:
    image: redis:7-alpine
    networks:
      - backend

networks:
  frontend:
    driver: bridge
  backend:
    driver: bridge
    internal: true

volumes:
  db_data:
```

### 6.12.4 Ejecución y verificación

#### Paso 1: Iniciar la arquitectura

```bash
$ cd laboratorio-redes
$ docker compose up -d
[+] Running 9/9
 ✔ Network laboratorio-redes_frontend  Created
 ✔ Network laboratorio-redes_backend   Created
 ✔ Container laboratorio-redes-cache-1 Started
 ✔ Container laboratorio-redes-db-1    Started
 ✔ Container laboratorio-redes-app-1   Started
 ✔ Container laboratorio-redes-app-2   Started
 ✔ Container laboratorio-redes-app-3   Started
 ✔ Container laboratorio-redes-app-4   Started
 ✔ Container laboratorio-redes-app-5   Started
 ✔ Container laboratorio-redes-lb-1    Started
```

#### Paso 2: Verificar las redes

```bash
$ docker network ls
NETWORK ID          NAME                            DRIVER    SCOPE
abc123def456        laboratorio-redes_frontend      bridge    local
def456abc123        laboratorio-redes_backend       bridge    local
```

```bash
$ docker network inspect laboratorio-redes_frontend \
  --format '{{range .Containers}}{{.Name}}: {{.IPv4Address}}{{"\n"}}{{end}}'
laboratorio-redes-lb-1: 172.20.0.2/16
laboratorio-redes-app-1: 172.20.0.3/16
laboratorio-redes-app-2: 172.20.0.4/16
laboratorio-redes-app-3: 172.20.0.5/16
laboratorio-redes-app-4: 172.20.0.6/16
laboratorio-redes-app-5: 172.20.0.7/16
```

```bash
$ docker network inspect laboratorio-redes_backend \
  --format '{{range .Containers}}{{.Name}}: {{.IPv4Address}}{{"\n"}}{{end}}'
laboratorio-redes-db-1: 172.21.0.2/16
laboratorio-redes-cache-1: 172.21.0.3/16
laboratorio-redes-app-1: 172.21.0.4/16
laboratorio-redes-app-2: 172.21.0.5/16
...
```

#### Paso 3: Demostrar resolución DNS

```bash
# Nginx resuelve "app" (una de las réplicas en frontend)
$ docker exec laboratorio-redes-lb-1 nslookup app
Name:      app
Address 1: 172.20.0.3
Address 2: 172.20.0.4
Address 3: 172.20.0.5
Address 4: 172.20.0.6
Address 5: 172.20.0.7

# Una app resuelve "db" y "cache" (en backend)
$ docker exec laboratorio-redes-app-1 nslookup db
Name:      db
Address 1: 172.21.0.2

$ docker exec laboratorio-redes-app-1 nslookup cache
Name:      cache
Address 1: 172.21.0.3

# MySQL no puede resolver "app" (está solo en backend)
$ docker exec laboratorio-redes-db-1 nslookup app
Server:    127.0.0.11
Address 1: 127.0.0.11 localhost
nslookup: can't resolve 'app'
```

#### Paso 4: Demostrar balanceo de carga

```bash
$ for i in $(seq 1 10); do curl -s http://localhost:8080/ | jq -r '.service'; done
laboratorio-redes-app-1
laboratorio-redes-app-3
laboratorio-redes-app-2
laboratorio-redes-app-5
laboratorio-redes-app-4
laboratorio-redes-app-1
laboratorio-redes-app-3
laboratorio-redes-app-2
laboratorio-redes-app-5
laboratorio-redes-app-4
```

Las peticiones se distribuyen entre las 5 réplicas. Nginx usa el DNS de Docker, y el DNS rota las IPs devueltas.

#### Paso 5: Demostrar aislamiento

```bash
# MySQL NO es accesible desde fuera del backend
$ curl http://localhost:3306
curl: (56) Recv failure: Connection reset by peer

# Las apps individuales NO son accesibles desde fuera (no tienen ports: expuestos)
$ curl http://localhost:5000
curl: (7) Failed to connect to localhost port 5000: Connection refused

# Solo Nginx (el balanceador) es accesible
$ curl -s http://localhost:8080/health
{"checks":{"database":true,"redis":true}}
```

#### Paso 6: Verificar contador de base de datos

```bash
$ for i in $(seq 1 20); do curl -s http://localhost:8080/ > /dev/null; done

$ curl -s http://localhost:8080/ | jq '.counters'
{
  "laboratorio-redes-app-1": 5,
  "laboratorio-redes-app-2": 4,
  "laboratorio-redes-app-3": 3,
  "laboratorio-redes-app-4": 4,
  "laboratorio-redes-app-5": 4
}
```

#### Paso 7: Verificar conectividad a Redis

```bash
$ curl -s -X POST http://localhost:8080/cache/clave \
  -H "Content-Type: application/json" \
  -d '{"value": "Hola desde Docker networks!"}'
{"clave":"Hola desde Docker networks!","status":"ok"}

$ curl -s http://localhost:8080/cache/clave
{"clave":"Hola desde Docker networks!"}
```

#### Paso 8: Troubleshooting con netshoot

Vamos a inspeccionar el tráfico en una app para ver las peticiones que recibe:

```bash
# En una terminal
$ docker run --rm --network container:laboratorio-redes-app-1 \
  nicolaka/netshoot tcpdump -i eth0 -n 'port 5000'
```

En otra terminal, haz una petición:

```bash
$ curl -s http://localhost:8080/ | jq '.service'
"laboratorio-redes-app-1"  # Cayó justo en app-1
```

Verás en la terminal de tcpdump:

```
14:32:10.123456 IP 172.20.0.2.45000 > 172.20.0.3.5000: Flags [S], seq ...
14:32:10.123567 IP 172.20.0.3.5000 > 172.20.0.2.45000: Flags [S.], seq ...
14:32:10.123678 IP 172.20.0.2.45000 > 172.20.0.3.5000: Flags [.], ack ...
14:32:10.124000 IP 172.20.0.2.45000 > 172.20.0.3.5000: Flags [P.], seq 1:150 ...
14:32:10.125000 IP 172.20.0.3.5000 > 172.20.0.2.45000: Flags [P.], seq 1:250 ...
```

Puedes trazar exactamente qué contenedor habla con cuál, cuándo, y qué datos se transmiten.

### 6.12.5 Destruir el laboratorio

```bash
$ docker compose down -v
[+] Running 8/8
 ✔ Container laboratorio-redes-lb-1    Removed
 ✔ Container laboratorio-redes-app-1   Removed
 ✔ Container laboratorio-redes-app-2   Removed
 ✔ Container laboratorio-redes-app-3   Removed
 ✔ Container laboratorio-redes-app-4   Removed
 ✔ Container laboratorio-redes-app-5   Removed
 ✔ Container laboratorio-redes-db-1    Removed
 ✔ Container laboratorio-redes-cache-1 Removed
 ✔ Network laboratorio-redes_frontend  Removed
 ✔ Network laboratorio-redes_backend   Removed
 ✔ Volume laboratorio-redes_db_data    Removed
```

`-v` elimina también los volúmenes (incluyendo los datos de MySQL).

### 6.12.6 Lecciones del laboratorio

1. **Las redes definidas por el usuario permiten arquitecturas multi-capa limpias**. Nada de IPs hardcodeadas.
2. **El DNS automático es habilitador de microservicios**. `app` resuelve a 5 IPs diferentes y Nginx balancea sin saberlo.
3. **El aislamiento es real**. MySQL y Redis son inaccesibles desde fuera. Solo los contenedores en `backend` pueden alcanzarlos.
4. **Conexión a múltiples redes**. Cada `app` tiene pie en `frontend` y `backend`. Esto es un patrón común: la app es el puente entre el mundo exterior y los datos internos.
5. **Sin Swarm, sin Kubernetes**. Solo con Docker y Compose tienes balanceo de carga, descubrimiento de servicios, y aislamiento.
6. **Troubleshooting integrado**. Con `netshoot` y `tcpdump` puedes depurar cada capa de red en tiempo real.

---

## 6.13 Resumen del Capítulo

### Los 7 drivers de red en Docker

| Driver | Aislamiento | Rendimiento | Multi-host | DNS Auto | Caso de uso principal |
|--------|-------------|-------------|------------|----------|-----------------------|
| **bridge** (defecto) | Parcial | Medio | No | No | Desarrollo, tests |
| **bridge** (usuario) | Bueno | Medio | No | Sí | Microservicios en un host |
| **host** | Ninguno | Máximo | No | No | Alto rendimiento, monitoreo |
| **overlay** | Bueno | Medio-bajo | Sí | Sí (Swarm) | Clústeres Swarm |
| **macvlan** | Físico | Máximo | Sí | No | Integración legacy, VLANs |
| **ipvlan** | Físico | Máximo | Sí | No | Cloud, SDN |
| **none** | Total | N/A | N/A | N/A | Seguridad máxima, batch |

### Reglas de oro

1. **Siempre usa redes definidas por el usuario** para cualquier proyecto serio. El bridge por defecto es para pruebas rápidas.
2. **No uses `--link`**. Está obsoleto. Usa nombres de contenedor y DNS automático.
3. **Publica solo lo necesario**. Si un contenedor no necesita ser accesible desde fuera, no publiques sus puertos.
4. **Aprovecha `internal: true`** para bases de datos y colas. No necesitan acceso a internet.
5. **El orden de las redes importa** en Compose. La primera red listada es la red por defecto del servicio (afecta a la ruta por defecto).
6. **Aprende `docker network inspect`**. Te sacará de más apuros que cualquier otra herramienta.
7. **Ten `netshoot` a mano**. Es la herramienta de troubleshooting más valiosa del ecosistema Docker.

### Lo que sigue

En el próximo capítulo abordaremos **almacenamiento en Docker**: volúmenes, bind mounts, tmpfs, drivers de almacenamiento, backup y restauración. Si las redes son el sistema nervioso de Docker, el almacenamiento es la memoria. Sin él, los contenedores son amnésicos.

---

*Fin del Capítulo 6: Redes en Docker.*
