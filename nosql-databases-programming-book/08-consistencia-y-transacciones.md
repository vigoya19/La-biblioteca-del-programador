# Capítulo 8: Consistencia Eventual y Transacciones Distribuidas

> "En un mundo distribuido, mantener a todos los servidores enterados del mismo suceso al mismo instante es una batalla perdida contra la velocidad de la luz. En su lugar, pactamos que todos eventualmente se pondrán de acuerdo."

El gran superpoder de las bases de datos NoSQL es su capacidad de escala horizontal gracias a la distribución de datos a lo largo de múltiples servidores. Sin embargo, este superpoder viene acompañado de una dura realidad física: **la sincronía perfecta en red es una ilusión**. 

Intentar forzar que múltiples servidores separados por miles de kilómetros geográficos mantengan un estado idéntico en tiempo real obliga al sistema a retener bloqueos de red masivos, disparando la latencia y destruyendo el rendimiento. En este capítulo, desmitificaremos cómo convivir con la **Consistencia Eventual**, cómo resolver conflictos de concurrencia y cómo implementar patrones de transacciones distribuidas avanzados como **Sagas** y el **Transactional Outbox**.

---

## 8.1 El Límite de la Sincronía: Consistencia Fuerte vs. Eventual

* **Consistencia Fuerte (Strong Consistency)**: Garantiza que cualquier lectura devuelva de forma inmediata la última escritura procesada. Obliga a congelar y bloquear todos los nodos réplicas durante la escritura para evitar lecturas de datos sucios.
* **Consistencia Eventual (Eventual Consistency)**: Permite que los nodos respondan al instante de forma asíncrona, aceptando que, durante unos pocos milisegundos (o segundos, en caso de fallos de red), diferentes usuarios en distintas partes del mundo vean datos ligeramente discrepantes. El compromiso físico es garantizar que, tras un periodo de inactividad de escrituras, todos los nodos convergerán y mostrarán exactamente el mismo valor.

---

## 8.2 Resolución de Conflictos: Relojes de Vector vs. LWW

Cuando múltiples clientes escriben de forma concurrente en diferentes nodos de una red con consistencia eventual, inevitablemente ocurrirán colisiones de datos. ¿Cómo decide el sistema cuál es el dato real?

### 1. Last-Write-Wins (LWW - El Último Gana)
Es el mecanismo más sencillo y común. Cada escritura viaja acompañada de la marca de tiempo (*timestamp*) física del servidor que la generó. El motor descarta los datos más viejos y retiene el dato con la marca de tiempo más nueva.
* **El Peligro (Clock Drift)**: Los relojes físicos de los servidores nunca están perfectamente sincronizados de forma microscópica, incluso usando servidores NTP. Un microsegundo de desajuste físico puede provocar que una escritura legítima sea descartada por una marca de tiempo desajustada de otro nodo.

### 2. Relojes de Vector (Vector Clocks)
Es un mecanismo lógico (no físico) utilizado en bases de datos como Cassandra y Dynamo para rastrear la causalidad de los eventos. Un Vector Clock es un array de tuplas `[Nodo, Contador]`:
$$\text{Vector} = [ (A, 1), (B, 3) ]$$
* Si el cliente A escribe un dato, el vector incrementa su contador en A.
* Si el cliente B lee ese dato y escribe una actualización, hereda el historial causal de A.
* Si dos actualizaciones llegan con contadores independientes que no son descendientes causales, el sistema detecta un **conflicto concurrente**, delegando la resolución lógica al cliente o disparando procesos de fusión inteligente (como CRDTs - Conflict-Free Replicated Data Types).

---

## 8.3 Transacciones en MongoDB: El Costo de ACID Distribuido

Aunque MongoDB nació como un sistema puramente NoSQL, desde su versión 4.0 admite **transacciones multi-documento ACID** con aislamiento *snapshot* a través de Replica Sets y Sharded Clusters.

* **Internals (Protocolo de Compromiso en Dos Fases - 2PC)**: Para garantizar que 3 documentos de colecciones separadas se actualicen juntos o fallen todos, MongoDB bloquea internamente los documentos afectados a nivel de motor de almacenamiento (WiredTiger), coordina la escritura entre los nodos réplicas y escribe logs de transacciones cruzadas.
* **El Costo**: Las transacciones multi-documento degradan el rendimiento de la base de datos de forma sustancial debido a la contención de bloqueos físicos. **Deben usarse con suma prudencia**, únicamente para operaciones críticas de negocio (como flujos financieros) y nunca para lecturas o escrituras masivas ordinarias.

---

## 8.4 El Patrón Saga: Transacciones Asíncronas de Larga Duración

En arquitecturas distribuidas orientadas a microservicios, una transacción comercial (por ejemplo, realizar un pedido) requiere la participación de múltiples microservicios y bases de datos independientes. Forzar transacciones ACID locales sincronizadas por red (como 2-Phase Commit distribuido) colapsaría el sistema si un servicio sufre latencia. 

Para resolver esto, implementamos el **Patrón Saga**: una secuencia de transacciones locales e independientes que se coordinan de forma asíncrona mediante mensajería.

```
       ┌──────────────┐         ┌───────────────┐         ┌────────────────┐
       │   Servicio   │         │   Servicio    │         │    Servicio    │
       │   Pedidos    │ ──────► │  Inventario   │ ──────► │  Facturación   │
       │ (Pedido Creado)        │ (Stock Reservado)       │  (Cobro Fallido)
       └──────────────┘         └───────────────┘         └───────┬────────┘
              ▲                         │                         │
              │  (Transacción de)       ▼  (Transacción de)       │
              └──( Compensación ) ◄─────┴─────────────────────────┘
                 ( Libera Stock )
```

* **Transacciones de Compensación (Compensating Transactions)**: En una Saga, si el paso 1 y el paso 2 tienen éxito, pero el paso 3 (cobrar con tarjeta de crédito) falla, la Saga no puede hacer un "ROLLBACK" físico en la base de datos del inventario del paso 2. En su lugar, el sistema debe disparar de forma explícita acciones compensatorias lógicas: *"Ejecutar script para devolver el stock al inventario"* y *"Cancelar orden de pedido en base de datos"*.
* **Modelos de Coordinación**:
  1. **Coreografía**: Cada servicio emite eventos locales y escucha eventos de otros servicios de forma totalmente desacoplada. Difícil de rastrear en sistemas masivos.
  2. **Orquestación**: Un servicio central (Orquestador de Sagas) actúa como el cerebro director de la orquesta, enviando comandos explícitos a cada servicio y gestionando de forma centralizada las compensaciones ante fallos lógicos.

---

## 8.5 El Patrón Transactional Outbox

Uno de los antipatrones más comunes y peligrosos en la arquitectura de software distribuido es la **Escritura Dual (Dual-Write)**. Ocurre cuando intentas hacer dos escrituras en sistemas externos diferentes dentro del mismo bloque de código:

```typescript
// ❌ ANTIPATRON: Escritura Dual
async function procesarPedido(pedido) {
  await pedidoModel.save(pedido); // 1. Escribir en Base de Datos local
  await messageBroker.publish('pedido-creado', pedido); // 2. Enviar a RabbitMQ/Kafka
}
```

* **El Riesgo**: Si la base de datos escribe con éxito, pero la red falla o RabbitMQ se cae un milisegundo después, tu base de datos tendrá el pedido guardado, pero el resto de tu ecosistema distribuido jamás se enterará de la venta (**Inconsistencia Catastrófica**).
* **La Solución (Transactional Outbox)**: En lugar de intentar escribir al broker de forma directa, guardas el pedido y el mensaje del evento (el *Outbox*) **dentro de la misma base de datos local utilizando una única transacción transaccional local inquebrantable**. 

Un proceso o worker secundario asíncrono e independiente lee continuamente la tabla o colección de Outbox (o escucha los logs de cambio mediante CDC - Change Data Capture) y publica el evento en el broker de forma fiable, garantizando la entrega al 100% (**Consistencia Eventual Garantizada**).

> [!NOTE]
> ### 📬 El Cartero con el Buzón de Salida Blindado
> 
> Visualicemos el dilema de la escritura dual en un hotel físico:
> 
> - **El Enfoque de Escritura Dual Fallido**: 
>   - Llega un huésped a la recepción. El recepcionista registra al huésped en su ordenador (Paso 1).
>   - Al mismo instante, levanta el teléfono para llamar al departamento de limpieza para que preparen la suite (Paso 2).
>   - Si de pronto la línea telefónica interna se cae y da ocupado, el huésped queda registrado en la base de datos del sistema, pero el mensaje de limpieza se pierde en el vacío. El huésped llegará a su habitación y estará sucia (**inconsistencia**).
> - **El Enfoque Transactional Outbox (Buzón de Salida)**:
>   - El recepcionista registra al huésped en su pantalla.
>   - Al mismo tiempo y como parte del mismo movimiento físico de su mano derecha, escribe una pequeña nota de papel autoadhesiva que dice *"Limpiar Suite del Huésped 102"* y la arroja dentro de una **Caja de Madera Blindada con Llave (la Colección Outbox)** que está atornillada al escritorio. Ambas acciones ocurren en un solo bloque atómico.
>   - Un **Cartero Dedicado (el Worker de CDC)** pasa por la recepción cada 5 segundos, abre la caja de madera con su llave, retira todas las notas físicas acumuladas, camina al departamento de limpieza y las entrega en mano una a una. 
>   - Si el teléfono está caído, el cartero simplemente espera a que vuelva la línea o las lleva caminando. Las notas de papel físicas jamás se perderán ni desaparecerán de la caja blindada. Se garantiza la entrega del mensaje final.

---

## 8.6 Implementación en TypeScript de Transactional Outbox

Implementemos una transacción segura con el patrón **Transactional Outbox** utilizando transacciones locales en MongoDB mediante Mongoose en TypeScript:

#### [OutboxModel.ts](file:///Users/andres/Documents/biblioteca/nosql-databases-programming-book/src/models/OutboxModel.ts)
```typescript
import { Schema, model, Document } from 'mongoose';

export interface IOutboxEvent extends Document {
  agregadoId: string;       // ID de la entidad afectada (ej: pedido_102)
  tipoEntidad: string;      // ej: "Pedido"
  nombreEvento: string;     // ej: "pedido.creado"
  payload: any;             // Contenido JSON del mensaje
  procesado: boolean;
  fechaCreacion: Date;
}

const OutboxEventSchema = new Schema<IOutboxEvent>({
  agregadoId: { type: String, required: true },
  tipoEntidad: { type: String, required: true },
  nombreEvento: { type: String, required: true },
  payload: { type: Schema.Types.Mixed, required: true },
  procesado: { type: Boolean, default: false, index: true },
  fechaCreacion: { type: Date, default: Date.now }
});

export const OutboxEvent = model<IOutboxEvent>('OutboxEvent', OutboxEventSchema);
```

#### [pedidoOutboxService.ts](file:///Users/andres/Documents/biblioteca/nosql-databases-programming-book/src/services/pedidoOutboxService.ts)
```typescript
import mongoose from 'mongoose';
import { Pedido } from '../models/EsquemaPedido';
import { OutboxEvent } from '../models/OutboxModel';

// Crear una orden de compra y su evento Outbox atómicamente en la misma sesión
export async function crearPedidoTransaccional(datosPedido: any): Promise<void> {
  const session = await mongoose.startSession();
  
  // Iniciamos una transacción local robusta en MongoDB
  session.startTransaction();

  try {
    // 1. Guardar el Pedido principal
    const [nuevoPedido] = await Pedido.create([datosPedido], { session });

    // 2. Guardar el evento en el buzón de salida (Outbox) dentro del mismo bloque físico
    const eventoOutbox = {
      agregadoId: nuevoPedido._id.toString(),
      tipoEntidad: 'Pedido',
      nombreEvento: 'pedido.creado',
      payload: {
        pedidoId: nuevoPedido._id,
        total: nuevoPedido.total,
        clienteEmail: nuevoPedido.cliente.email
      }
    };

    await OutboxEvent.create([eventoOutbox], { session });

    // 3. Confirmar la transacción atómica local
    await session.commitTransaction();
  } catch (error) {
    // Deshacer cualquier escritura local si algo falla en el proceso
    await session.abortTransaction();
    throw error;
  } finally {
    session.endSession();
  }
}
```

---

## 8.7 Deep Dive: Anatomía Matemática de los CRDTs (G-Counters y LWW-Sets)

Para lograr una consistencia eventual garantizada sin necesidad de costosos bloqueos de red concurrentes coordinados (como el algoritmo Paxos o Raft), las bases de datos distribuidas de vanguardia (como Riak, Redis Enterprise o Apache Cassandra) implementan **Tipos de Datos Replicados Libres de Conflictos (CRDTs - Conflict-Free Replicated Data Types)**.

> [!NOTE]
> ### 🗒️ La Analogía de las Listas del Campamento y las Notas de Tareas Tachadas
> 
> Visualicemos el funcionamiento lógico de los CRDTs sin asustarnos por las matemáticas:
> 
> - **El problema de contar concurrentemente (G-Counter)**:
>   - Imagina que estás coordinando un gran campamento infantil con **3 Coordinadores (los Servidores A, B y C)**.
>   - Si intentan llevar una sola libreta central compartida y gritar los números en caliente por walkie-talkie, habrá interferencias de red y acabarán sumando mal.
>   - **La Solución**: Le entregas a cada coordinador una hoja con **3 Columnas de firmas (A, B y C)**.
>     - El Coordinador A solo escribe firmas en la columna A. El B solo en la B. El C solo en la C.
>     - Si el Coordinador A recluta a un niño, cambia su columna de 4 a 5. Su hoja mide `[5, 0, 0]`.
>     - De forma paralela y sin hablar con nadie, el Coordinador C recluta a dos niños. Su columna pasa a 2. Su hoja mide `[0, 0, 2]`.
>     - **La Fusión (Merge)**: Cuando se encuentran en el comedor, juntan sus hojas. En lugar de sumar todos los números a ciegas (lo que duplicaría datos), comparan celda por celda y **se quedan con el número más alto (el máximo)**.
>     - Al fusionar `[5, 0, 0]` y `[0, 0, 2]`, la hoja combinada resulta en `[5, 0, 2]`. La suma total es $5 + 0 + 2 = 7$ niños. El orden en que se encuentren no altera el resultado final, y si juntan sus hojas dos veces seguidas no se altera nada (**Idempotencia**).
> - **El problema de Añadir/Quitar de una lista (LWW-Element-Set)**:
>   - Tienes una lista de compras de víveres compartida con tu pareja.
>   - Llevas dos libretas: **La Libreta Verde (Añadidos)** y **La Libreta Roja (Borrados)**.
>   - Si quieres comprar Manzanas, escribes en la Libreta Verde: *"Manzanas (21:00)"*.
>   - Si decides cancelarlo, en lugar de borrar con goma de borrar, escribes en la Libreta Roja: *"Manzanas (21:05)"*.
>   - Al unirse en el supermercado, juntan todas sus libretas verdes y rojas.
>   - Para saber si compran manzanas, buscan en ambas libretas. Como la marca de tiempo de la Libreta Roja (21:05) es más nueva que la de la Libreta Verde (21:00), la manzana está oficialmente cancelada. La fusión es limpia, sin necesidad de que hablen en tiempo real por teléfono.

### 1. El Fundamento Matemático: Semilretículo de Fusión (Join-Semilattice)
Para que un tipo de datos distribuido pueda fusionar actualizaciones concurrentes procedentes de diferentes servidores lejanos sin errores de inconsistencia, el operador de fusión interna (**$\sqcup$ - Merge**) de la estructura de datos debe formar un **Semilretículo de Fusión Acotado**. Esto exige que la función de fusión cumpla tres propiedades algebraicas estrictas:


1. **Asociatividad**: $(A \sqcup B) \sqcup C = A \sqcup (B \sqcup C)$. (El orden de agrupación en red de los paquetes de datos no altera el resultado final).
2. **Conmutatividad**: $A \sqcup B = B \sqcup A$. (El orden de llegada de los eventos físicos por el cable de red no importa).
3. **Idempotencia**: $A \sqcup A = A$. (La entrega duplicada de un mismo mensaje de red no altera el valor del dato, neutralizando reintentos caóticos).

### 2. G-Counter (Grow-Only Counter - Contador de Solo Incremento)
Un contador distribuido simple en red no puede fusionarse simplemente sumando valores (si el Servidor A sumó +1 y el Servidor B sumó +1 concurrente, un simple broadcast sumaría +2 incorrectamente de forma repetida).

Un **G-Counter** de estado se define matemáticamente como un vector $V$ de longitud $N$ (donde $N$ es el número de servidores réplicas en el cluster):
* Cada servidor $i$ es dueño único de su celda $V[i]$.
* Para incrementar el contador, el servidor $i$ ejecuta localmente:
  $$V[i] = V[i] + 1$$
* El valor total del contador se calcula sumando todos los elementos del vector:
  $$\text{Valor Total} = \sum_{k=1}^{N} V[k]$$
* **El Operador de Fusión ($\sqcup$)**: Cuando el Servidor A y el Servidor B sincronizan sus vectores por red, el fusionador calcula el **máximo elemento por elemento** de ambos vectores:
  $$V_{\text{merged}}[k] = \max(V_A[k], V_B[k]) \quad \forall k \in [1, N]$$
  Esta operación es perfectamente asociativa, conmutativa e idempotente, logrando convergencia matemática absoluta al instante.

### 3. LWW-Element-Set (Conjunto del Último Gana)
Un conjunto que permite añadir y eliminar elementos concurrentemente se modela en un **LWW-Element-Set** manteniendo dos conjuntos internos de tuplas con marcas de tiempo físicas: el **Add Set ($A$)** y el **Remove Set ($R$)**:

* **Añadir** elemento $x$ con marca de tiempo $t$:
  $$A = A \cup \{(x, t)\}$$
* **Eliminar** elemento $x$ con marca de tiempo $t$:
  $$R = R \cup \{(x, t)\}$$
* **Operación de Búsqueda (Lookup)**: ¿Pertenece el elemento $x$ al conjunto actual?
  * El elemento $x$ es miembro activo si y solo si existe en el conjunto de adiciones ($A$) y, o bien no existe en el conjunto de eliminaciones ($R$), o bien su marca de tiempo de adición más reciente ($t_a$) es estrictamente superior a su marca de tiempo de eliminación más reciente ($t_r$):
    $$\exists (x, t_a) \in A \quad \text{tal que} \quad (\forall (x, t_r) \in R, t_a > t_r)$$
* **Operador de Fusión ($\sqcup$)**: Fusionar dos conjuntos LWW es simplemente calcular la unión matemática básica de sus respectivos subconjuntos internos, eliminando conflictos por completo:
  $$A_{\text{merged}} = A_1 \cup A_2 \quad \text{y} \quad R_{\text{merged}} = R_1 \cup R_2$$

---

## Resumen del Capítulo


* En sistemas distribuidos NoSQL masivos, la **consistencia eventual** es una decisión arquitectónica obligatoria para erradicar las latencias físicas de red.
* Conflictos concurrentes se resuelven mediante el uso de marcas físicas con políticas **LWW** o lógicas causales mediante **Vector Clocks**.
* El **Patrón Saga** modela flujos de transacciones asíncronas de larga duración coordinadas mediante Coreografías u Orquestaciones con pasos explícitos de **compensación lógica**.
* El patrón **Transactional Outbox** erradica el antipatrón de escrituras duales, persistiendo el evento de integración y la entidad de negocio en una sola transacción local inquebrantable para su posterior transmisión.

En el próximo capítulo, entraremos de lleno a la optimización analítica de accesos analizando la **Indexación Avanzada**, la regla ESR en MongoDB y la integración robusta con motores de búsqueda dedicados como **Elasticsearch** y **OpenSearch**.

---

[← Capítulo anterior (Capítulo 7)](07-modelado-avanzado.md) | [Inicio (README.md)](README.md) | [Capítulo siguiente (Capítulo 9) →](09-indexacion-y-busqueda.md)
