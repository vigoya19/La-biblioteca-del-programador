# Capítulo 11: Monitoreo, Profiling y Optimización de Consultas

> "Si no puedes medir el comportamiento en caliente de tus bases de datos, no estás administrando una infraestructura de software; estás cruzando los dedos y esperando que no ocurra un colapso."

Poner una base de datos NoSQL en producción es solo el inicio del ciclo de vida de un sistema de alto rendimiento. En el instante en que tu software comienza a recibir tráfico real y a acumular millones de registros históricos en disco, los pequeños errores de diseño que en fase de desarrollo tardaban 1 milisegundo en ejecutarse comienzan a retrasar peticiones, inflar las facturas de tu proveedor de Cloud y, eventualmente, colapsar tu servidor por completo. 

El arte de mantener bases de datos robustas no se basa en adivinar; se basa en diagnosticar científicamente. En este capítulo, aprenderemos a utilizar el comando `.explain()` en MongoDB para diagnosticar consultas lentas, optimizaremos los costos de aprovisionamiento de capacidad en DynamoDB y estudiaremos las políticas de desalojo de memoria en Redis para prevenir catástrofes de falta de memoria (Out-Of-Memory).

---

## 11.1 Profiling de Consultas Lentas en MongoDB con `.explain()`

Cuando una consulta tarda más de lo esperado en MongoDB, la herramienta de diagnóstico fundamental de primer nivel es el método **`.explain("executionStats")`**. Este comando le ordena al motor de MongoDB que ejecute la consulta en frío de forma real, registre minuciosamente las métricas de rendimiento y devuelva un informe analítico detallado del plan de ejecución físico.

```
       ┌────────────────────────────────────────────────────────┐
       │   Comando: db.usuarios.find(query).explain("exec...")  │
       └───────────────────────────┬────────────────────────────┘
                                   │
                                   ▼
       ┌────────────────────────────────────────────────────────┐
       │   Informe Analítico: COLLSCAN (❌) vs. IXSCAN (✅)       │
       └────────────────────────────────────────────────────────┘
```

### Métricas Críticas a Analizar en el JSON de Explain:
1. **`stage`**: Define cómo accedió el motor a los datos.
   * `COLLSCAN` (❌ Peligro): Indica que MongoDB escaneó de forma secuencial cada archivo del disco duro buscando coincidencias. Falta un índice de forma crítica.
   * `IXSCAN` (✅ Óptimo): Indica que MongoDB utilizó un árbol indexado ordenado para saltar directamente al dato buscado.
   * `FETCH`: Indica que tras consultar el índice, el motor saltó al disco a buscar los campos adicionales del documento.
   * `PROJSORT` / `SORT`: Indica que el ordenamiento de los resultados se realizó en memoria RAM, un síntoma de que el índice compuesto no sigue la regla ESR.
2. **`totalKeysExamined`**: Número de llaves de índices físicos que leyó el motor.
3. **`totalDocsExamined`**: Número de documentos físicos de disco que se cargaron a memoria RAM.
4. **`nReturned`**: Cantidad de documentos devueltos finalmente al usuario.

### La Ecuación de Oro del Performance:
Para garantizar que tu base de datos esté perfectamente optimizada y consuma la menor cantidad de procesador y memoria física del disco, debes acercar tu sistema a la siguiente proporción matemática:

$$\frac{\text{totalDocsExamined}}{\text{nReturned}} \approx 1$$

* Si la división devuelve un valor muy superior a 1 (por ejemplo, examinas 100,000 documentos en disco para devolver solo 5 resultados finales), significa que tu índice no es lo suficientemente selectivo y estás desperdiciando ciclos de lectura física en disco de forma masiva.

---

## 11.2 Optimización de Costos y Capacidades en DynamoDB

A diferencia de las bases de datos auto-administradas en servidores propios, Amazon DynamoDB cobra de forma directa en base al volumen de datos leídos y escritos por segundo utilizando dos métricas clave:

* **RCU (Read Capacity Unit - Unidad de Capacidad de Lectura)**: Representa una lectura fuertemente consistente por segundo, o dos lecturas eventualmente consistentes por segundo, para un documento de hasta **4 KB** de tamaño.
* **WCU (Write Capacity Unit - Unidad de Capacidad de Escritura)**: Representa una escritura por segundo para un documento de hasta **1 KB** de tamaño.

### El Peligro del Sobredimensionamiento y Throttling:
Si calculas mal tus capacidades y configuras 10 RCUs por segundo, pero tu aplicación de pronto recibe peticiones concurrentes equivalentes a 50 RCUs por segundo, DynamoDB rechazará las conexiones de tus usuarios de forma inmediata arrojando la excepción **`ProvisionedThroughputExceededException` (Throttling)**.
* **Soluciones**:
  1. Activar el modelo de precios **On-Demand (Bajo Demanda)** en entornos con cargas de tráfico impredecibles y picos extremos de uso.
  2. Implementar **DynamoDB Accelerator (DAX)**: una caché en memoria dedicada de altísima velocidad que intercepta lecturas repetitivas aliviando el consumo de RCU de tus tablas físicas en producción.

---

## 11.3 Prevención de Out-Of-Memory (OOM) en Redis

Redis almacena todos sus datos exclusivamente en la memoria RAM del servidor. Al ser la RAM un recurso físico limitado y costoso, si tu aplicación inyecta millones de llaves de forma descontrolada sin límite de expiración, el servidor colapsará por completo arrojando errores de falta de memoria (OOM).

Para blindar Redis, configuramos en el archivo `redis.conf` el límite de memoria máximo (`maxmemory`) y seleccionamos una **Política de Desalojo de Llaves (Eviction Policy)** en caliente:

1. **`volatile-lru` (Least Recently Used)**: Desaloja (elimina) las llaves usadas con menor frecuencia del servidor, pero **únicamente** aquellas que cuenten con un tiempo de expiración TTL configurado.
2. **`allkeys-lru`**: Elimina cualquier llave menos usada recientemente del servidor de forma indiscriminada para abrir espacio físico, sin importar si tiene TTL o no. Ideal para configuraciones dedicadas a caché.
3. **`allkeys-lfu` (Least Frequently Used)**: Analiza un contador de frecuencia de uso por cada llave, eliminando aquellas llaves cuya frecuencia de consultas diarias sea ínfima.
4. **`noeviction`** (Predeterminada y peligrosa): Redis no elimina nada. Si la RAM se llena, cualquier nueva escritura arrojará un error catastrófico del sistema, paralizando las escrituras de tu backend.

> [!NOTE]
> ### 🩺 El Estetoscopio del Mecánico en la Fila de Montaje
> 
> Visualicemos el monitoreo de rendimiento en la vida real:
> 
> - **El explain() de MongoDB** es equivalente a llevar tu coche deportivo al taller porque pierde caballos de fuerza:
>   - En lugar de desarmar todo el motor pieza por pieza a ciegas (COLLSCAN manual), el mecánico conecta un **Computador de Diagnóstico al Puerto OBD2 (el comando `.explain()`)**.
>   - La pantalla le dice de inmediato: *"El filtro de aire está obstruido y el motor succiona 10,000 litros de aire para producir 1 solo caballo de fuerza"* (un ratio insalubre de docs examined vs returned). Limpias el filtro (creas el índice preciso) y el coche vuela.
> - **Las Capacidades RCU/WCU de DynamoDB** son equivalentes a **Las Cabinas de Peaje de una Autopista Privada**:
>   - Cada coche (una petición del cliente) paga un peaje según su tamaño en kilos.
>   - Un utilitario pequeño (un documento JSON menor a 4 KB) paga una moneda (1 RCU). Un camión cisterna gigante de 20 toneladas (un JSON pesado de 20 KB) debe pagar 5 monedas físicas (5 RCUs) porque desgasta más el asfalto.
>   - Si aprovisionas peajes únicamente para 10 monedas por segundo y llega un convoy de 10 camiones pesados al mismo instante, se generará un **atasco de tráfico kilométrico (Throttling)** en la entrada de la autopista. Debes habilitar peajes dinámicos bajo demanda o desviar los coches repetitivos a un helipuerto de alta velocidad de memoria (DAX).

---

## 11.4 Ejecución de Profiling explain() mediante TypeScript

Implementemos una consulta en Mongoose en TypeScript que ejecute de forma programática un `.explain()` y analice si el plan de ejecución físico arroja un preocupante `COLLSCAN` para disparar alertas automáticas de infraestructura:

#### [profilerService.ts](file:///Users/andres/Documents/biblioteca/nosql-databases-programming-book/src/services/profilerService.ts)
```typescript
import { Usuario } from '../models/UsuarioModel';

interface AnalisisConsulta {
  stage: string;
  docsExaminados: number;
  clavesExaminadas: number;
  tiempoEjecucionMs: number;
  esPeligrosa: boolean;
}

// Analizar la eficiencia de una búsqueda específica en producción
export async function perfilarBusquedaUsuarios(filtro: any): Promise<AnalisisConsulta> {
  // Ordenamos a Mongoose ejecutar el explain en nivel de estadísticas de ejecución
  const explicacion = await Usuario.find(filtro).explain('executionStats');
  
  // Extraemos las estadísticas del primer stage del plan de ejecución
  const stats = explicacion.executionStats;
  const stageFisico = stats.executionStages.stage;
  const docsExaminados = stats.totalDocsExamined;
  const clavesExaminadas = stats.totalKeysExamined;
  const tiempoEjecucionMs = stats.executionTimeMillis;

  // Una consulta es peligrosa si realiza un COLLSCAN en disco o si examina demasiados documentos en vano
  const esPeligrosa = stageFisico === 'COLLSCAN' || (docsExaminados > 1000 && docsExaminados > stats.nReturned * 5);

  if (esPeligrosa) {
    console.warn(`[ALERTA-INFRAESTRUCTURA] Consulta lenta detectada! 
      Filtro: ${JSON.stringify(filtro)} 
      Stage: ${stageFisico} 
      Docs Examinados: ${docsExaminados} 
      Tiempo: ${tiempoEjecucionMs}ms`);
  }

  return {
    stage: stageFisico,
    docsExaminados,
    clavesExaminadas,
    tiempoEjecucionMs,
    esPeligrosa
  };
}
```

---

  return {
    stage: stageFisico,
    docsExaminados,
    clavesExaminadas,
    tiempoEjecucionMs,
    esPeligrosa
  };
}
```

---

## 11.5 Deep Dive: WiredTiger Cache Eviction Bottlenecks y Ticket Starvation

En MongoDB, el motor de almacenamiento subyacente de bajo nivel es **WiredTiger**. Cuando administras clusters de misión crítica que manejan miles de escrituras concurrentes por segundo, monitorear el uso de CPU o el espacio en disco no es suficiente. Debes vigilar los internals del flujo de memoria RAM de WiredTiger:

> [!NOTE]
> ### 🏬 La Analogía del Restaurante de Aforo Limitado y los Platos Sucios
> 
> Visualicemos el bloqueo físico de escrituras por saturación de caché en disco:
> 
> - **El Motor WiredTiger (la Caché en RAM)** es equivalente a un **Restaurante de Moda** con un aforo limitado de exactamente **128 Mesas**:
>   - Para entrar a comer (realizar una operación de lectura o escritura), cada cliente debe tomar obligatoriamente un **Ticket Numerado (un Ticket de WiredTiger)** en el dispensador de la puerta. Solo hay 128 tickets físicos en circulación.
>   - **El Desalojo de Caché (Eviction)** es equivalente al **Equipo de Limpieza del Restaurante**:
>     - Cuando un cliente termina de comer (los datos se escriben en memoria RAM), el equipo de limpieza limpia rápidamente la mesa, recoge los platos y los lleva a lavar al lavavajillas del sótano (escribe de forma asíncrona la página sucia de memoria al disco físico SSD). La mesa queda libre y el ticket se devuelve al dispensador de la puerta.
>   - **El Cuello de Botella y Bloqueo General (Ticket Starvation)**:
>     - De pronto, el lavavajillas del sótano se avería o se satura (tus discos SSD sufren un pico de latencia física o se agotan las IOPS de tu servidor Cloud).
>     - El equipo de limpieza tarda ahora 10 minutos en lavar un solo plato. Las mesas se acumulan con platos sucios y los comensales no pueden retirarse.
>     - Como las 128 mesas están ocupadas y sucias, el dispensador de la puerta **se queda sin tickets físicos (Ticket Starvation)**.
>     - Fuera del restaurante, una multitud colosal de miles de personas hambrientas (las conexiones de red de tu API Express backend) se acumula en la calle esperando un ticket. El restaurante colapsa y la base de datos se congela por completo en producción.
>     - **La Solución**: No sirve de nada gritarle a los cajeros; debes reparar la velocidad del lavavajillas del sótano (incrementar el rendimiento de lectura/escritura de disco SSD y sus IOPS).

### 1. El Cuello de Botella de Desalojo (WiredTiger Cache Eviction)
MongoDB mantiene las páginas de datos calientes y los índices en una caché de memoria RAM gestionada por WiredTiger (por defecto, configurada al $50\%$ de la RAM total del servidor menos 1 GB).
* Cuando modificas o insertas documentos, WiredTiger marca las páginas de memoria RAM como **"Dirty Pages" (Páginas Sucias)**.
* De forma paralela, unos hilos de fondo llamados **Eviction Threads** monitorizan continuamente la caché. En cuanto el porcentaje de páginas sucias supera el $20\%$ de la capacidad de la caché, estos hilos empiezan a escribir de fondo los cambios al disco físico para vaciar y limpiar la RAM.
* Si el rendimiento de escritura de tu disco físico (IOPS) se satura o sufre latencia, los hilos de desalojo no pueden escribir al ritmo que llegan los nuevos datos. Las páginas sucias saturan la caché de memoria RAM.

### 2. Hambruna de Tickets (Ticket Starvation)
Para proteger la base de datos de un desbordamiento catastrófico de memoria RAM, WiredTiger implementa un mecanismo de control de concurrencia basado en **Tickets**:
* Por defecto, el motor provee **128 Tickets de Lectura** y **128 Tickets de Escritura** simultáneos.
* Cada consulta entrante de tu backend debe adquirir un ticket libre antes de poder entrar a operar en la caché de WiredTiger.
* Si la caché de memoria RAM se satura por fallos en el desalojo a disco, los hilos de desalojo retienen y congelan los tickets activos de forma interna para forzar la detención de nuevas escrituras y evitar el colapso del servidor.
* **El Síntoma**: Las conexiones activas del backend se disparan exponencialmente esperando un ticket libre, la latencia de tu API pasa de 2ms a 30,000ms en segundos, y el comando `db.serverStatus().wiredTiger.concurrentTransactions` muestra que los tickets disponibles han llegado a 0.

---

## Resumen del Capítulo


* Mantener bases de datos rápidas requiere el uso continuo de **`.explain("executionStats")`** para erradicar las búsquedas secuenciales en disco (`COLLSCAN`).
* La eficiencia óptima de indexación se logra cuando la relación entre **documentos examinados y devueltos converge a 1**.
* El consumo en AWS DynamoDB se optimiza minimizando el tamaño físico de los objetos JSON leídos (RCUs) y escritos (WCUs) e implementando aceleradores **DAX**.
* Redis se protege contra caídas catastróficas de falta de memoria configurando límites estrictos (`maxmemory`) y seleccionando políticas de desalojo inteligente como **`volatile-lru`** o **`allkeys-lfu`**.

En el próximo y último capítulo, uniremos todos los hilos de este libro diseñando e implementando un sistema real enterprise que combina bases documentales, clave-valor, memoria y grafos en una arquitectura de **Persistencia Políglota (Polyglot Persistence)**.

---

[← Capítulo anterior (Capítulo 10)](10-seguridad.md) | [Inicio (README.md)](README.md) | [Capítulo siguiente (Capítulo 12) →](12-proyecto-practico.md)
