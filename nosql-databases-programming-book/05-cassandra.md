# Capítulo 5: Apache Cassandra y el Almacenamiento Wide-Column

> "En una arquitectura máster-esclavo clásica, la caída del líder colapsa el reino. En una arquitectura masterless descentralizada, todos los nodos son plebeyos y reyes al mismo tiempo: el sistema sobrevive sin importar quién caiga."

Cuando el volumen de datos de tu aplicación escala a petabytes globales y necesitas procesar cientos de miles de escrituras por segundo distribuidas geográficamente a través de múltiples centros de datos físicos sin admitir un solo segundo de inactividad, las arquitecturas con nodos maestros únicos (como los primarios de MongoDB o las bases de datos SQL tradicionales) colapsan. 

**Apache Cassandra** fue desarrollada originalmente por Facebook (para impulsar su bandeja de entrada de chat) e inspirada en dos pilares del diseño distribuido: el documento **Dynamo Paper** de Amazon (para su anillo de distribución masterless) y **Bigtable** de Google (para su modelo de datos wide-column y su motor físico LSM-Tree). Cassandra es una base de datos distribuida linealmente escalable, tolerante a fallos y sin un punto único de fallo (SPOF).

---

## 5.1 La Arquitectura Descentralizada Masterless

A diferencia de otros sistemas de bases de datos que dependen de un nodo líder para coordinar escrituras y replicar datos a esclavos pasivos, Cassandra adopta una arquitectura **Masterless (sin maestro)**:

```
                    ┌──────────────┐
             ┌─────►│  Nodo 1 (A)  │◄─────┐
             │      └──────┬───────┘      │
             ▼             │ (Gossip)     ▼
      ┌──────────────┐     │       ┌──────────────┐
      │  Nodo 3 (C)  │◄────┼──────►│  Nodo 2 (B)  │
      └──────────────┘             └──────────────┘
```

* **Simetría Absoluta**: Todos los nodos del cluster son idénticos y realizan exactamente las mismas funciones. Cualquier nodo puede actuar como un **Nodo Coordinador** para interceptar una petición del cliente. El cliente se conecta a cualquier nodo aleatorio de la red y este enruta la consulta hacia los nodos físicos responsables de almacenar esa porción de datos exacta.
* **Protocolo de Gossip (Chisme)**: La salud del cluster, el estado de los nodos y los metadatos de topología se comunican de forma asíncrona y continua entre los nodos mediante un protocolo par a par (P2P) inspirado en los rumores humanos. Cada segundo, cada nodo intercambia información con algunos vecinos aleatorios, logrando que el estado del cluster converja en toda la red en pocos segundos.

---

## 5.2 El Anillo de Hashing Consistente y Replicación

Para repartir millones de filas físicas entre decenas o cientos de nodos de forma homogénea y dinámica sin requerir un servidor centralizador de metadatos, Cassandra emplea el **Hashing Consistente (Consistent Hashing)**:

* **El Anillo de Tokens**: Cassandra organiza el espacio de almacenamiento lógico como un anillo circular tridimensional que va del token $-2^{63}$ al token $2^{63}-1$. Cada nodo físico del cluster se adueña de un rango numérico de este anillo (o múltiples rangos mediante **Virtual Nodes - vnodes**).
* **Particionamiento**: Cuando llega una fila, Cassandra toma el valor de su clave de partición, lo procesa mediante su algoritmo de particionamiento rápido (`Murmur3Partitioner`) para obtener un valor hash entero de 64 bits (el Token) y lo hace girar sobre el anillo circular en sentido de las agujas del reloj. El primer nodo físico cuyo rango de token sea mayor o igual al token de la fila se convierte en el responsable primario del dato.
* **Factor de Replicación (RF)**: Define en cuántos nodos físicos diferentes se copiará cada fila para evitar pérdida de datos si fallan las máquinas. Si $RF = 3$, el nodo coordinador entregará la fila al nodo primario correspondiente del anillo y de forma asíncrona a los siguientes dos nodos físicos situados consecutivamente en el anillo circular.

---

## 5.3 LSM-Trees: ¿Por qué las Escrituras son Ultra-Rápidas?

En las bases de datos relacionales tradicionales (que usan árboles B-Tree en disco), escribir un dato implica modificar una página física de disco existente. Esto requiere que las cabezas físicas de lectura/escritura den saltos caóticos buscando sectores aleatorios en el disco duro (**Random I/O**), lo que ralentiza y colapsa el rendimiento. 

Cassandra elimina el Random I/O por completo implementando un motor de almacenamiento secuencial tipo **LSM-Tree (Log-Structured Merge-Tree)**:

```
[ Petición de Escritura ] 
        │
        ├──► [ 1. Commit Log ] (Escritura secuencial rápida en disco físico)
        │
        └──► [ 2. Memtable ]   (Estructura volátil rápida en memoria RAM)
                 │
                 ▼ (Cuando Memtable se llena)
             [ 3. SSTable ]    (Escritura de bloque secuencial inmutable en disco)
```

1. **Commit Log (Disco)**: Cuando llega una escritura, Cassandra la registra de inmediato de forma secuencial al final de un archivo append-only en el disco físico llamado Commit Log. Al ser secuencial, el disco escribe a velocidades teóricas máximas sin realizar saltos aleatorios. Este archivo sirve únicamente para recuperación ante apagones.
2. **Memtable (RAM)**: Al mismo tiempo, el dato se inyecta en una estructura ordenada de acceso rápido en memoria RAM llamada Memtable. En cuanto estas dos operaciones terminan, Cassandra le responde "Éxito" al cliente, logrando escrituras con latencias de microsegundos:
   $$O(1)$$
3. **SSTables (Sorted String Tables)**: Cuando la Memtable se llena o se supera un umbral de tiempo, se vuelca (*flush*) al disco físico de forma secuencial como un archivo ordenado e inmutable llamado **SSTable**. Como las SSTables son inmutables (nunca se reescriben en caliente), no hay contención de bloqueos ni corrupción de archivos.
4. **Bloom Filters**: Como existen decenas de SSTables en disco debido a los constantes volcados, buscar una fila de lectura obligaría al sistema a abrir todos los archivos de disco. Cassandra optimiza esto con **Bloom Filters**: estructuras probabilísticas ultrarrápidas en memoria RAM que le indican al motor, con 0% de falsos negativos: *"Este dato definitivamente no está en esta SSTable"*, permitiendo saltarse la lectura física del archivo.
5. **Compactación (Compaction)**: Periódicamente, un hilo secundario une y fusiona múltiples SSTables de fondo en un nuevo archivo limpio, eliminando duplicados antiguos y aplicando los borrados marcados por banderas especiales llamadas **Tombstones** (lápidas).

> [!NOTE]
> ### ⭕ El Hula-Hoop de Reparto Masivo y los Escribanos Súper Rápidos
> 
> Visualicemos Cassandra en tu centro logístico:
> 
> - **El Anillo circular (Consistent Hashing)** es un gigantesco **Hula-Hoop circular suspendido en el aire** dividido en secciones numeradas. Tienes 5 buzones físicos colgando en diferentes puntos del aro.
> - Cuando llega una carta, la arrojas por un túnel que le aplica un giro y un número de fuerza (el algoritmo hash). La carta recorre el hula-hoop circular y cae por gravedad en el primer buzón que encuentre en su camino en sentido del reloj. Si añades un nuevo buzón colgado del hula-hoop, solo tienes que redistribuir un pequeño segmento de cartas de la zona vecina, sin alterar al resto de las cartas de los buzones lejanos (**Escalabilidad lineal**).
> - **El Escribano LSM-Tree**: Imagina que la oficina de correos recibe miles de cartas de cambios de domicilio por segundo. 
>   - En lugar de ir a los estantes físicos y buscar la carpeta de cartón de cada persona para borrar la dirección antigua y escribir la nueva (Random I/O lento de SQL), el director contrata a un **Escribano en la Recepción** con una libreta en su escritorio (la **Memtable** en RAM). El escribano anota rápidamente los cambios según llegan.
>   - Cuando llena la libreta, el escribano le pone una grapa y la arroja en una caja fuerte de metal en el fondo de la oficina (**SSTable inmutable**). No borra nada del pasado; si hay un cambio de domicilio para "Juan", simplemente escribe una nueva hoja más actualizada.
>   - **El problema de la lectura**: Si Juan viene a preguntar cuál es su dirección actual, el director tendría que registrar miles de libretas archivadas en la caja fuerte. Para evitarlo, coloca a un **Conserje con Memoria Prodigiosa** (el **Bloom Filter**) en la puerta. El conserje mira el nombre de Juan y dice: *"Te aseguro que en las libretas 3, 5 y 8 no está el nombre de Juan; no pierdas tiempo abriéndolas"*. El director solo abre la libreta 12, que es la más nueva y donde reside el dato correcto.

---

## 5.4 Consistencia Tuneable: La Ecuación Matemática del Quórum

Cassandra es un sistema fundamentalmente AP (según el Teorema de CAP), pero te otorga el superpoder de elegir el nivel de consistencia exacto en caliente por cada consulta de lectura y escritura individual mediante el concepto de **Consistencia Tuneable**:

Definamos las variables del sistema:
* $N$: Factor de Replicación (en cuántos nodos físicos se almacena la fila).
* $W$: Nivel de Consistencia de Escritura (a cuántos nodos réplica debe confirmar el coordinador con éxito antes de responder al cliente).
* $R$: Nivel de Consistencia de Lectura (a cuántos nodos réplica debe consultar el coordinador para verificar y contrastar que la firma de datos sea idéntica antes de responder al cliente).

### La Ecuación del Quórum Estricto (Consistencia Fuerte):
Para garantizar que cualquier lectura devuelva siempre el dato de la última escritura con éxito, debes cumplir la siguiente inecuación matemática:

$$R + W > N$$

* Si cumples la ecuación, se garantiza que la intersección entre los nodos consultados para leer ($R$) y los nodos modificados al escribir ($W$) tiene al menos un nodo en común. Ese nodo en común actuará como el portador de la verdad. Cassandra comparará las marcas de tiempo físicas de los nodos comunes, detectará cuál es la más reciente y aplicará un proceso automático de reparación de fondo (**Read Repair**) en los nodos desactualizados de forma transparente.

---

## 5.5 CQL y Conexión en TypeScript con cassandra-driver

En Cassandra **no existen los JOINs ni las llaves foráneas**. El modelado de datos se realiza de forma estricta orientada a las consultas (**Query-Driven Modeling**). Si una pantalla de tu app muestra usuarios por país, creas la tabla `usuarios_por_pais`. Si otra muestra usuarios por edad, creas otra tabla `usuarios_por_edad`, duplicando la información de forma estratégica.

Implementemos una conexión robusta y una consulta utilizando el controlador oficial `cassandra-driver` en TypeScript:

#### [cassandraClient.ts](file:///Users/andres/Documents/biblioteca/nosql-databases-programming-book/src/clients/cassandraClient.ts)
```typescript
import { Client, auth } from 'cassandra-driver';

// 1. Inicializar la conexión distribuida de Cassandra especificando los puntos de contacto
export const cassandraClient = new Client({
  contactPoints: ['127.0.0.1'], // Direcciones IP de los nodos semilla
  localDataCenter: 'datacenter1', // Nombre de tu Datacenter local
  keyspace: 'biblioteca_nosql',  // Equivalente a base de datos
  authProvider: new auth.PlainTextAuthProvider('cassandra_user', 'super_secure_pass_123')
});
```

#### [sensorService.ts](file:///Users/andres/Documents/biblioteca/nosql-databases-programming-book/src/services/sensorService.ts)
```typescript
import { cassandraClient } from './clients/cassandraClient';

export interface LecturaSensor {
  sensorId: string;
  timestamp: Date;
  temperatura: number;
}

// 2. Registrar datos de telemetría IoT de forma masiva
export async function registrarLecturaSensor(lectura: LecturaSensor): Promise<void> {
  // CQL utiliza llaves de partición (sensorId) y columnas de clustering (timestamp) para ordenar en disco
  const query = `
    INSERT INTO lecturas_sensores (sensor_id, fecha_registro, temperatura)
    VALUES (?, ?, ?)
  `;
  
  const params = [lectura.sensorId, lectura.timestamp, lectura.temperatura];
  
  // Ejecutamos con nivel de consistencia QUORUM local para mayor seguridad
  await cassandraClient.execute(query, params, { 
    prepare: true, 
    consistency: 10 // Representa LOCAL_QUORUM en el cassandra-driver
  });
}

// 3. Obtener el historial cronológico de un sensor físico en rangos de tiempo
export async function obtenerHistorialSensor(sensorId: string, desde: Date): Promise<LecturaSensor[]> {
  const query = `
    SELECT sensor_id, fecha_registro, temperatura 
    FROM lecturas_sensores 
    WHERE sensor_id = ? AND fecha_registro > ?
  `;
  
  const params = [sensorId, desde];
  const resultado = await cassandraClient.execute(query, params, { prepare: true });
  
  return resultado.rows.map(row => ({
    sensorId: row.sensor_id,
    timestamp: row.fecha_registro,
    temperatura: row.temperatura
  }));
}
```

---

## 5.6 Deep Dive: Phi Accrual Failure Detector y Hinted Handoffs

En un cluster distribuido de Cassandra con cientos de nodos físicos, las fallas son la norma diaria y no la excepción. Dos mecanismos microscópicos garantizan la resiliencia absoluta del cluster:

> [!NOTE]
> ### 🏥 La Analogía del Médico de Guardia Suspicaz y el Mensaje de Recepción
> 
> Visualicemos el monitoreo de fallas en red y la recuperación ante cortes de servidores:
> 
> - **El Failure Detector Clásico (Fijo)** es equivalente a un **Médico de Guardia Cuadrículado**:
>   - El médico vigila a un paciente y tiene una regla fija: *"El paciente debe parpadear exactamente cada 5 segundos. Si tarda 5.1 segundos, declararé su fallecimiento inmediatamente"*. Si de pronto una mota de polvo entra en el ojo del paciente y este tarda 6 segundos en parpadear (una congestión de red), el médico inicia los trámites de defunción erróneamente, causando un caos en el hospital.
> - **El Phi Accrual Failure Detector de Cassandra** es equivalente a un **Médico Sabio y Probabilístico**:
>   - El médico monitoriza el historial real de los últimos 1,000 parpadeos del paciente. Sabe que el promedio es de 5 segundos, pero con ligeras variaciones de milisegundos según el clima.
>   - Si el paciente tarda en parpadear, el médico no entra en pánico; simplemente incrementa su **nivel de sospecha interna numérica ($\Phi$ - Phi)**. Con cada milisegundo extra de retraso, la sospecha sube: $\Phi = 1$, luego $\Phi = 3$, luego $\Phi = 5$.
>   - Si $\Phi$ supera el umbral de 8 o 10, el médico sabe matemáticamente que la probabilidad de que el paciente esté vivo es de una entre cien millones. Declara el fallo con absoluta certeza, adaptándose al comportamiento biológico natural.
> - **Hinted Handoffs (Notas de Resguardo)**: Es equivalente a que un **Mensajero (el Nodo Coordinador)** llegue a entregar un paquete a la habitación del paciente A, pero descubre que el paciente se fue a hacer un examen médico (el nodo está caído).
>   - En lugar de tirar el paquete a la basura o cancelar la entrega completa, el mensajero camina a la **Recepción de Enfermería del Sótano (el disco local del Coordinador)**, deja el paquete y escribe un post-it autoadhesivo (**un Hint**): *"Entregar este medicamento al Paciente A en cuanto regrese"*.
>   - En cuanto los altavoces del hospital anuncian que el Paciente A ha regresado a su habitación (el chisme del protocolo Gossip), la enfermera recoge el paquete y se lo entrega en mano, logrando consistencia final sin bloquear al mensajero en la entrada.

### 1. El Algoritmo del Phi Accrual Failure Detector
En sistemas distribuidos clásicos, para saber si una máquina se ha caído se utiliza un ping simple con un tiempo de espera fijo (por ejemplo, *ping cada 5 segundos, si tarda más, asumir caída*). 
 
* **El Problema**: Las redes sufren congestión física momentánea. Un tiempo de espera demasiado corto provoca falsas alarmas (nodos marcados como caídos que en realidad estaban vivos pero lentos, lo que inicia redistribuciones de datos innecesarias que saturan la red). Un tiempo de espera muy largo retrasa la detección de fallas reales, degradando el rendimiento de los clientes.
* **La Solución**: Cassandra implementa el **Phi Accrual Failure Detector** (un detector probabilístico basado en sospecha).
  * En lugar de devolver una respuesta binaria (Vivo/Muerto), el detector calcula un valor continuo de sospecha numérica llamado **$\Phi$ (Phi)**.
  * Cassandra monitoriza continuamente y guarda en una ventana deslizante de memoria los intervalos de tiempo en milisegundos entre los últimos latidos de corazón (*heartbeats*) de cada nodo réplica.
  * Utilizando una distribución de probabilidad normal basada en el historial de latencia real de la red, calcula la probabilidad de que el siguiente latido llegue tarde:
    $$\Phi = -\log_{10}(P_{\text{late}}(t - t_{\text{last}}))$$
    donde $t_{\text{last}}$ es la marca del último latido recibido.
  * Si $\Phi = 8$, la probabilidad de cometer un error de falso positivo es de $1$ entre $10^8$ (una entre cien millones). Si $\Phi = 10$ o $12$, el umbral de seguridad es extremo. Esto permite que Cassandra se adapte dinámicamente a la salud microscópica de la red en tiempo real.

### 2. Hinted Handoffs (Escrituras con Réplicas Inactivas)
¿Qué ocurre si ejecutas una escritura con $RF = 3$ y uno de los nodos réplica responsables está caído o inaccesible temporalmente?
* Si tu nivel de consistencia exigido es `ONE` o `LOCAL_QUORUM`, la escritura puede procesarse con éxito en los dos nodos réplica restantes.
* **La Nota de Resguardo (Hint)**: Para evitar dejar al tercer nodo permanentemente desactualizado, el **Nodo Coordinador** que interceptó la petición toma el cambio físico de datos, lo empaqueta en una estructura llamada **Hint** (un resguardo temporal con marca de tiempo) y lo escribe de forma secuencial en su propio disco local.
* **La Entrega asíncrona**: El coordinador monitoriza los chismes del protocolo Gossip. En cuanto el Gossip anuncia que el nodo réplica caído se ha recuperado y su detector $\Phi$ vuelve a estar en verde, el coordinador abre su directorio local de hints, extrae las transacciones acumuladas y las inyecta secuencialmente en el nodo réplica recién recuperado, restaurando la consistencia en el cluster de forma 100% asíncrona y transparente.

---

## Resumen del Capítulo


* Apache Cassandra es una base de datos **Wide-Column masterless descentralizada** diseñada para escalabilidad horizontal lineal masiva y disponibilidad total.
* Utiliza un **anillo de hashing consistente** para repartir los tokens de datos de forma dinámica entre nodos, evitando un servidor central de metadatos.
* Su motor **LSM-Tree** transforma operaciones complejas de escritura aleatoria en escrituras físicas secuenciales ultra-rápidas mediante Commit Logs, Memtables y SSTables inmutables.
* El arquitecto de sistemas puede modular en caliente la latencia y la robustez del sistema aplicando la ecuación matemática de **Consistencia Tuneable** ($R + W > N$).

En el próximo capítulo, ingresaremos al mundo de la alta conectividad de relaciones complejas: **Neo4j** y las bases de datos de **Grafos**, desmitificando el Cypher Query Language y la arquitectura Index-Free Adjacency.

---

[← Capítulo anterior (Capítulo 4)](04-redis.md) | [Inicio (README.md)](README.md) | [Capítulo siguiente (Capítulo 6) →](06-neo4j.md)
