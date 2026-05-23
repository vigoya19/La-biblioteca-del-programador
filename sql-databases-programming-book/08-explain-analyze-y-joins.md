# Capítulo 8: Optimización de Consultas con EXPLAIN ANALYZE

> "No adivines por qué tu base de datos relacional va lenta. El optimizador de consultas de PostgreSQL no es una caja negra impenetrable; es una máquina de cálculo estadístico altamente predecible. Deja de especular, ejecuta EXPLAIN ANALYZE y lee los Rayos X de tu código físico."

Cuando un desarrollador junior se enfrenta a una consulta lenta en producción, su primer instinto suele ser agregar índices de forma desordenada a todas las columnas involucradas o culpar al hardware. Sin embargo, para un ingeniero senior, cada optimización comienza con el mismo comando: **`EXPLAIN ANALYZE`**.

Esta herramienta interroga directamente al planificador de consultas de PostgreSQL (el *Planner/Optimizer* que vimos en el Capítulo 1), obligándolo a revelar el plan físico de operaciones que diseñó para extraer los datos de los bloques del disco duro. En este capítulo, aprenderemos a leer e interpretar estos planes paso a paso, analizaremos los diferentes tipos de lecturas físicas (Seq Scan vs. Index Scan), estudiaremos los algoritmos de unión física (`Nested Loop`, `Hash Join` y `Merge Join`), y crearemos una herramienta en TypeScript capaz de monitorear programáticamente consultas lentas y detectar patrones ineficientes.

---

## 8.1 Anatomía de EXPLAIN y EXPLAIN ANALYZE

Es vital comprender la diferencia radical entre estas dos sentencias:
* **`EXPLAIN <consulta>`**: Genera una estimación teórica del plan de ejecución basada en las estadísticas acumuladas en el catálogo de PostgreSQL (`pg_statistic`). **No ejecuta la consulta física en disco**. Es instantáneo y seguro para usar en producción con cualquier consulta de escritura.
* **`EXPLAIN ANALYZE <consulta>`**: **Ejecuta la consulta físicamente en la base de datos** y mide los microsegundos reales consumidos en cada nodo del plan, comparándolos con las estimaciones previas. 
  > [!WARNING]
  > Como `EXPLAIN ANALYZE` ejecuta la consulta, si lo corres sobre un `DELETE FROM usuarios;` o un `UPDATE`, los registros se borrarán o modificarán físicamente en disco. Para auditar escrituras de forma segura, siempre debes envolver la prueba dentro de una transacción que culmine en `ROLLBACK`.

### Las Métricas Clave de un Nodo:
Un plan se lee de adentro hacia afuera (de los nodos más anidados hacia el nodo raíz superior):
```text
Seq Scan on usuarios  (cost=0.00..35.50 rows=10 width=244) (actual time=0.015..0.042 rows=10 loops=1)
```
* **`cost=0.00..35.50`**: Estimación de coste arbitraria del motor. El primer número es el coste de arranque (tiempo antes de devolver la primera fila); el segundo es el coste de finalización total.
* **`rows=10`**: Cantidad estimada de filas que devolverá este nodo.
* **`actual time=0.015..0.042`**: Tiempo real medido en milisegundos para arrancar y finalizar este nodo.
* **`loops=1`**: Cuántas veces se ejecutó este nodo físico en bucle. El tiempo real debe multiplicarse por los bucles para obtener el consumo real consolidado.

---

## 8.2 Métodos de Escaneo Físico (Table Scans)

El optimizador selecciona cómo leer los datos del disco duro según la selectividad del filtro y los índices disponibles:

### 1. Sequential Scan (Seq Scan)
* **Operación**: Lee físicamente todo el archivo de la tabla en disco, página por página y tupla por tupla, buscando las filas que cumplen el filtro.
* **Cuándo es óptimo**: En tablas pequeñas (donde leer secuencialmente es más rápido que cargar índices en RAM) o cuando se solicita más del 20-30% de la tabla completa.

### 2. Index Scan
* **Operación**: Busca en el archivo del índice (B-Tree) los bloques que corresponden a la clave de búsqueda, extrae los punteros físicos de fila (**CTID**) y va al archivo de la tabla a extraer la fila completa.
* **Cuándo es óptimo**: Consultas muy selectivas que devuelven un volumen bajo de registros.

### 3. Index Only Scan
* **Operación**: Lee únicamente el archivo del índice. Si las columnas solicitadas en el `SELECT` forman parte de la definición del índice, PostgreSQL no necesita ir al archivo físico de la tabla a buscar más datos.
* **Cuándo es óptimo**: Altísimo rendimiento, reduce las lecturas físicas de disco casi a cero.

### 4. Bitmap Index Scan / Bitmap Heap Scan
* **Operación**: Paso intermedio. El motor escanea el índice y crea un mapa de bits en RAM indicando exactamente qué páginas de disco contienen las tuplas buscadas (`Bitmap Index Scan`). Luego, lee esas páginas del disco duro de forma ordenada secuencialmente, evitando saltos aleatorios de cabezal (`Bitmap Heap Scan`).

---

## 8.3 Algoritmos Físicos de JOIN

Cuando unes dos tablas, el optimizador decide dinámicamente qué algoritmo utilizar:

| Algoritmo de JOIN | Funcionamiento | Cuándo se usa |
| :--- | :--- | :--- |
| **Nested Loop** | Para cada fila de la Tabla A (externa), busca en bucle las filas coincidentes en la Tabla B (interna). | Óptimo si una tabla es pequeña y la otra tiene la columna del JOIN indexada. |
| **Hash Join** | Lee la Tabla B y crea una tabla hash en RAM (`WorkMem`). Luego escanea la Tabla A y busca coincidencias en la tabla hash en microsegundos. | Óptimo para unir tablas medianas/grandes sin índices ordenados comunes en caliente. |
| **Merge Join** | Ordena ambas tablas por la clave de unión (o usa un índice B-Tree ya ordenado) y las escanea de forma paralela en una sola pasada. | Óptimo cuando ambas tablas son gigantes pero ya están ordenadas por la columna de unión. |

---

> [!NOTE]
> ### 🩺 El Escáner de Rayos X de la Fila de Peajes (Explain Analyze)
> 
> Entendamos cómo funcionan los escaneos físicos y las uniones JOIN utilizando analogías del mundo real:
> 
> - **El Seq Scan (El Escaneo de Fila Completa de Supermercado)**:
>   - Imagina que entras a un supermercado buscando un bote de salsa picante exótica de mango.
>   - En lugar de ir al pasillo correcto, decides caminar por todos los pasillos del establecimiento, uno por uno, revisando cada estante de arriba a abajo hasta encontrarlo (**Sequential Scan**).
>   - Si el supermercado es pequeño (una tiendita de esquina), este método es rápido. Pero si es un hipermercado de dos pisos, es una pérdida de tiempo absurda.
> 
> - **El Index Scan (Ir con el Número de Pasillo exacto)**:
>   - En lugar de caminar a ciegas, vas al terminal de información en la entrada, buscas "Salsa de mango" y la pantalla te dice: *"Pasillo 5, Estante B, Sección 3"* (el **Índice B-Tree**).
>   - Caminas directamente al Pasillo 5 (**Index Scan**), tomas el bote del estante y vas a pagar. Redujiste tu esfuerzo físico al mínimo absoluto.
> 
> - **El Index Only Scan (La Compra Rápida en el Terminal de Entrada)**:
>   - Quieres saber únicamente el precio del bote de salsa de mango.
>   - Vas al terminal de información en la entrada y el buscador de pantalla te dice de inmediato: *"Salsa de mango -> Precio: $4.50 USD"*.
>   - Como obtuviste la información del precio directamente en la terminal (**Index Only Scan**), te retiras del supermercado inmediatamente sin necesidad de caminar físicamente hacia los pasillos internos.
> 
> - **El Nested Loop JOIN (El Cartero y las Cartas en los Buzones)**:
>   - Un cartero lleva un saco con 10 cartas dirigidas a una calle específica (**Tabla Externa**).
>   - Para cada carta que saca de su saco, camina por la calle buscando el número de casa coincidente en las fachadas para entregarla (**Bucle Interno**).
>   - Si tiene pocas cartas, este proceso es extremadamente eficiente. Pero si el saco tiene 10,000 cartas y debe recorrer 10,000 casas en cada una, el cartero colapsará por cansancio.
> 
> - **El Hash Join JOIN (El Organizador de Pulseras de Colores)**:
>   - Tienes que emparejar a un grupo de 1,000 corredores con sus 1,000 chips de tiempo correspondientes.
>   - En lugar de buscar corredor por corredor, montas una mesa en la entrada con cubículos numerados por el último dígito del chip (**Tabla Hash en RAM**).
>   - Colocas los chips rápidamente en sus cubículos. Luego, a medida que llega cada corredor, miras su número, vas al cubículo exacto en la mesa y le entregas su chip en menos de 2 segundos. Organizar la mesa costó un poco de esfuerzo inicial, pero la velocidad de entrega posterior fue instantánea.

---

## 8.4 Implementación en TypeScript de un Analizador de Planes de Consulta

A continuación, implementaremos una utilidad avanzada en TypeScript que ejecuta consultas críticas de nuestro sistema financiero utilizando `EXPLAIN (FORMAT JSON, ANALYZE)` para interpretar programáticamente el plan físico arrojado por PostgreSQL. Esto nos permitirá detectar y alertar de forma automática si una consulta en producción está realizando un ineficiente **Sequential Scan** sobre una tabla crítica:

#### [analizadorConsultas.ts](file:///Users/andres/Documents/biblioteca/sql-databases-programming-book/src/services/analizadorConsultas.ts)
```typescript
import { dbPool } from '../clients/dbClient';

export interface AlertaOptimizacion {
  tipoAlerta: 'INLINE_SEQ_SCAN' | 'HIGH_COST' | 'SLOW_EXECUTION';
  tabla: string;
  detalle: string;
  tiempoTotalMs: number;
}

// Ejecuta una consulta envolviéndola en un análisis dinámico del optimizador físico
export async function auditarPlanConsulta(
  sqlConsulta: string,
  parametros: any[] = []
): Promise<{ planExitoso: boolean; alertas: AlertaOptimizacion[]; tiempoTotalMs: number }> {
  
  const cliente = await dbPool.connect();
  const alertas: AlertaOptimizacion[] = [];

  try {
    // 1. Anteponer EXPLAIN (FORMAT JSON, ANALYZE) para obtener la telemetría en JSON
    const sqlExplain = `EXPLAIN (FORMAT JSON, ANALYZE) ${sqlConsulta}`;
    const resultado = await cliente.query(sqlExplain, parametros);
    
    // PostgreSQL devuelve el plan estructurado como un array anidado de JSON
    const planCompleto = resultado.rows[0]['QUERY PLAN'];
    const nodoRaiz = planCompleto[0].Plan;
    const tiempoTotalMs = parseFloat(planCompleto[0]['Execution Time']);

    // 2. Analizar recursivamente los nodos buscando cuellos de botella
    analizarNodoRecursivo(nodoRaiz, alertas, tiempoTotalMs);

    return {
      planExitoso: true,
      alertas,
      tiempoTotalMs
    };

  } catch (error: any) {
    console.error('Error al auditar el plan de la consulta:', error.message);
    return { planExitoso: false, alertas: [], tiempoTotalMs: 0 };
  } finally {
    cliente.release();
  }
}

// Analizador recursivo del árbol físico de operaciones
function analizarNodoRecursivo(
  nodo: any,
  alertas: AlertaOptimizacion[],
  tiempoTotalMs: number
) {
  const tipoNodo = nodo['Node Type'];
  const relacion = nodo['Relation Name'] || 'Desconocida';
  const costoTotal = nodo['Total Cost'];

  // Criterio 1: Detectar Sequential Scans en tablas que superen un tamaño lógico considerable
  // (Omitimos tablas pequeñas o de configuración donde un Seq Scan es natural)
  if (tipoNodo === 'Seq Scan' && nodo['Plan Rows'] > 1000) {
    alertas.push({
      tipoAlerta: 'INLINE_SEQ_SCAN',
      tabla: relacion,
      detalle: `Se detectó un escaneo secuencial completo de la tabla '${relacion}' con estimación de ${nodo['Plan Rows']} filas. Se recomienda crear un índice B-Tree adecuado.`,
      tiempoTotalMs
    });
  }

  // Criterio 2: Alertar de costes estimados alarmantes
  if (costoTotal > 5000) {
    alertas.push({
      tipoAlerta: 'HIGH_COST',
      tabla: relacion,
      detalle: `El coste total estimado de este nodo (${costoTotal}) excede los límites recomendados. Evalúe refactorizar la lógica o las uniones de la consulta.`,
      tiempoTotalMs
    });
  }

  // Analizar recursivamente los sub-nodos hijos (planes anidados) si existen
  if (nodo.Plans && Array.isArray(nodo.Plans)) {
    for (const subNodo of nodo.Plans) {
      analizarNodoRecursivo(subNodo, alertas, tiempoTotalMs);
    }
  }
}
```

---

## Resumen del Capítulo

* **`EXPLAIN`** estima teóricamente el plan de consulta basándose en estadísticas sin ejecutarla, mientras que **`EXPLAIN ANALYZE`** ejecuta físicamente la consulta en disco midiendo milisegundos reales.
* Los **Seq Scans** leen todo el archivo de la tabla en disco y son ineficientes en tablas grandes. Los **Index Scans** van directo al grano usando el índice B-Tree, y los **Index Only Scans** eliminan las lecturas del archivo base al resolver todo en el índice.
* Los **Nested Loops** destacan al emparejar conjuntos de datos muy pequeños con columnas indexadas. Los **Hash Joins** dominan en conjuntos de datos medianos/grandes mediante tablas hash en RAM, y los **Merge Joins** son la opción estrella para tablas masivas previamente ordenadas.
* El análisis programático de planes mediante JSON nos permite construir salvaguardas continuas en nuestros entornos de desarrollo para evitar subidas de código ineficiente a producción.

En el próximo capítulo, aprenderemos cómo romper los límites de escalabilidad física de una sola tabla mediante el estudio de **Particionamiento y Sharding de Tablas** en profundidad.

---

[← Capítulo anterior (Capítulo 7)](07-modelado-y-normalizacion.md) | [Inicio (README.md)](README.md) | [Capítulo siguiente (Capítulo 9) →](09-particionamiento-y-sharding.md)
