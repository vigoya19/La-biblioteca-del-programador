# Capítulo 2: Anatomía de la Indexación (B-Trees, Hash y GIN)

> "Un índice no hace que la base de datos sea mágicamente más rápida; hace que el motor lea menos datos de disco para responder exactamente a la misma pregunta."

En el desarrollo de software relacional, los índices son la herramienta principal para acelerar las lecturas de base de datos. Sin embargo, muchos desarrolladores asumen que colocar índices en todas las columnas resolverá milagrosamente sus problemas de latencia. Esto es un grave error: **los índices no son gratuitos**. Cada índice añadido consume espacio físico en disco, satura la memoria caché RAM y penaliza sustancialmente la velocidad de las escrituras (`INSERT`, `UPDATE`, `DELETE`) al obligar al motor a reestructurar árboles de disco en cada transacción.

En este capítulo, desmitificaremos la física de las estructuras de indexación, analizando desde la mecánica de búsqueda binaria de los árboles **B-Tree** hasta los índices **Hash** y la potencia de los **Índices Invertidos GIN** para búsquedas complejas de texto plano y objetos JSONB.

---

## 2.1 El Árbol Balanceado B-Tree en Detalle

El tipo de índice predeterminado y más potente en las bases de datos relacionales es el **B-Tree (Árbol B de Búsqueda Balanceado)**. 

```
                     [ Nodo Raíz (Root) ]
                          /        \
             [ Nodo Interno ]     [ Nodo Interno ]
                 /      \             /      \
             [ Hoja ]  [ Hoja ]   [ Hoja ]  [ Hoja ] ◄── Punteros físicos a disco (TID)
```

Un B-Tree organiza las claves del índice en una estructura de árbol multinivel con las siguientes características mecánicas:

1. **Auto-Balanceado**: El árbol garantiza de forma estricta que todas las páginas hojas (las que residen al final del árbol y contienen los punteros físicos a los datos reales) se encuentren exactamente a la **misma profundidad vertical (altura)**. Esto asegura tiempos de acceso predecibles e idénticos para cualquier clave.
2. **Estructura de las Páginas**: Cada nodo del árbol es una página física de disco (generalmente de 8 KB). Un nodo contiene un conjunto ordenado de claves y punteros que dirigen hacia las páginas hijas de abajo.
3. **Punteros Físicos (CTID)**: Las páginas hojas no contienen la fila completa del usuario. Almacenan únicamente el valor indexado (por ejemplo, el `email`) junto al **TID (Tuple Identifier / CTID)**, un puntero físico de bytes que le dice al motor: *"Este dato reside en la página de disco número 540, offset número 12"*.
4. **Complejidad Logarítmica**: Al realizar búsquedas binarias saltando de nodo en nodo, la complejidad de acceso para localizar cualquier registro se mantiene constante en tiempo logarítmico:
   $$O(\log N)$$
   Para una tabla de 10,000,000 de filas, un B-Tree localiza el registro en apenas 3 o 4 lecturas de disco.

### Internals Físicos: Page Splits, High Keys y Fillfactor
Cuando insertamos registros de forma masiva sobre un índice B-Tree, las páginas del árbol (nodos de 8KB) se van llenando.
* **Page Split (División de Página)**: Si intentas insertar una clave en una página física de índice que ya ha agotado sus 8KB, PostgreSQL debe dividir esa página en dos de forma atómica en disco. Para hacerlo, crea una página nueva, traslada la mitad superior de las claves a ella, inyecta la nueva clave, y escribe un registro llamado **High Key** (el valor límite máximo que puede albergar la página izquierda) para guiar futuras búsquedas. Este proceso causa una penalización severa en el rendimiento de escritura por I/O aleatoria en disco y reestructuración de árboles en RAM.
* **El Parámetro `fillfactor`**: Podemos optimizar esto al crear el índice configurando el `fillfactor` (un porcentaje entre $10$ y $100$, por defecto es $90$ para índices). Si configuramos `WITH (fillfactor = 70)`, PostgreSQL reservará a propósito un 30% de espacio libre en cada página del índice durante su creación. Este "colchón" evita la ocurrencia de Page Splits tempranos cuando los registros adyacentes sufran inserciones concurrentes en producción.

---

## 2.2 Índices Hash: Velocidad Constante $O(1)$

PostgreSQL admite índices basados en tablas de hashing:
* **Mecánica**: El motor pasa la clave por una función hash y localiza el valor directamente de forma constante:
  $$O(1)$$
* **Limitación Crítica**: Son sumamente veloces, pero **únicamente** sirven para filtros de igualdad exacta (`=`). Al no almacenar los datos ordenados en un árbol lineal, son completamente inútiles para consultas de rangos (`>`, `<`, `BETWEEN`), ordenamientos (`ORDER BY`), o búsquedas parciales (`LIKE 'an%'`). En producción, casi siempre se prefiere B-Tree por su versatilidad.

---

## 2.3 Índices GIN (Generalized Inverted Index)

Cuando almacenas documentos JSONB flexibles o textos largos y necesitas buscar de forma predictiva palabras o comprobar si una clave existe dentro de un array, un B-Tree regular falla porque solo puede indexar el valor del bloque JSON completo. 

Para resolver esto, empleamos **GIN (Índice Invertido Generalizado)**:
* **Mecánica**: En lugar de mapear una fila a sus valores, un índice GIN **invierte la relación**. Toma cada elemento individual dentro de un array o cada campo dentro de un JSONB, y mapea ese elemento específico a una lista de todos los IDs de filas donde se menciona.
* **Caso de Uso**: Ideal para acelerar búsquedas de pertenencia de arrays o filtros dinámicos en esquemas JSONB nativos de PostgreSQL.

---

## 2.4 Índices Especializados: GiST, SP-GiST y BRIN

Para resolver problemas que escapan a las búsquedas tradicionales, PostgreSQL ofrece motores de indexación sumamente avanzados:

### 1. GiST (Generalized Search Tree)
* **Cómo funciona**: No es un árbol rígido; es una plantilla de indexación balanceada que permite estructurar árboles de búsqueda sobre tipos de datos geométricos, rangos, o documentos jerárquicos complejos.
* **Caso de uso**: Búsquedas geográficas o mapas de proximidad (usando la extensión PostGIS) o para evitar solapamientos temporales mediante restricciones de exclusión.

### 2. SP-GiST (Space-Partitioned GiST)
* **Cómo funciona**: Diseñado para dividir el espacio de búsqueda en cuadrantes o estructuras de árbol de búsqueda no balanceadas (como árboles Quadtree o Tries).
* **Caso de uso**: Coordenadas geográficas de puntos en mapas bidimensionales o prefijos telefónicos.

### 3. BRIN (Block Range Index)
* **Cómo funciona**: En lugar de indexar fila por fila en disco, un índice BRIN **asume que los datos están almacenados de forma física ordenada cronológicamente en disco** (por ejemplo, una columna `creado_at` en una serie temporal). BRIN solo guarda una línea de metadatos por cada bloque físico de 1MB de páginas en disco, apuntando exclusivamente al **valor mínimo y máximo** contenido en ese bloque de 1MB.
* **Pros**: Es extremadamente diminuto. Ocupa un $1\%$ del tamaño físico de un índice B-Tree, permitiendo indexar tablas de miles de millones de filas consumiendo apenas unos cuantos kilobytes en RAM.
* **Contras**: Únicamente es útil si la tabla está perfectamente ordenada físicamente en disco por el campo clave.

---

## 2.5 El Visibility Map (VM) y el Secreto del Index Only Scan

Como vimos, un **Index Scan** requiere dos lecturas: buscar en el índice B-Tree el CTID, e ir al bloque Heap físico de la tabla a recuperar las columnas. Un **Index Only Scan** promete omitir el segundo viaje al disco leyendo las columnas directamente de los nodos hoja del índice.

Sin embargo, debido a que PostgreSQL implementa MVCC (Multiversión), la tupla en el índice no almacena el estado de visibilidad transaccional (`xmin`, `xmax`). ¿Cómo sabe el Executor si la fila indexada está visible para la transacción activa actual sin ir a leer la cabecera de la tupla física en el disco?

### El Visibility Map (Mapa de Visibilidad)
PostgreSQL almacena un archivo auxiliar en disco por cada tabla llamado **Visibility Map (VM)**.
* **Estructura**: Es un mapa de bits sumamente compacto donde cada bloque físico de 8KB de la tabla tiene un bit asignado.
* **El bit "all-visible"**: Si el bit es **`1`**, significa que todas las tuplas contenidas en esa página de 8KB son completamente visibles para cualquier transacción activa del sistema (porque ya son antiguas y están consolidadas).
* **Mecánica del Index Only Scan**:
  1. El Executor lee la clave en el índice B-Tree.
  2. Consulta el bit correspondiente de la página en el **Visibility Map** (que reside precargado en la memoria RAM ultrarrápida).
  3. Si el bit es **`1` (all-visible)**, el motor devuelve el valor de forma inmediata, **evitando por completo el costoso viaje al Heap de datos en disco duro**.
  4. Si el bit es **`0`**, el motor se ve obligado a realizar un fallback de I/O de disco para leer la página física de la tabla y comprobar si la tupla sigue viva.
* **El rol de VACUUM**: El proceso asíncrono de autovacuum es el encargado exclusivo de escanear las páginas, consolidar transacciones y encender los bits `all-visible` en el Visibility Map. Si tu base de datos no corre `VACUUM` regularmente, el Visibility Map se llenará de bits `0` y tus Index Only Scans se degradarán en Seq Scans encubiertos.

---

> [!NOTE]
> ### 🔍 El Fichero de Tarjetas y el Índice Alfabético Temático del Libro
> 
> Visualicemos la física de la indexación:
> 
> - **El COLLSCAN (Seq Scan sin Índices)**: Es equivalente a caminar por una biblioteca de 500,000 libros y abrirlos uno por uno, hoja por hoja, buscando si en algún párrafo se menciona al autor *"Edgar Codd"*. Tardarás meses.
> - **El Índice B-Tree**: Es equivalente al **Fichero de Tarjetas Alfabetizado de la Recepción**:
>   - Llegas y ves 3 cajones principales de madera: *Cajón A-G (Raíz)*, *Cajón H-O*, *Cajón P-Z*.
>   - Abres el primer cajón (Nodo Interno). Rebuscas y saltas directo a la tarjeta de *Codd (Nodo Hoja)*.
>   - La tarjeta de papel tiene una coordenada física escrita a lápiz que dice: *"Estantería 4, Pasillo 3, Altura 2" (el CTID)*. Caminas al estante exacto y recoges el libro. Tardaste 10 segundos.
> - **El Índice Invertido GIN**: Es equivalente al **Índice Temático al Final de un Libro de Medicina**:
>   - Si buscas en qué páginas del libro se menciona la palabra *"Aspirina"*, no lees el libro completo. Vas al índice al final, buscas "Aspirina", y ves una lista ordenada: *"Páginas: 12, 45, 98, 120"*.
>   - Saltas directamente a esas 4 páginas y lees. GIN descompone internamente los elementos complejos (arrays, palabras, JSONs) y crea este listado invertido de páginas físicas en el disco de forma transparente.

---

## 2.4 Creación de Índices Concurrency Zero-Downtime

En PostgreSQL, si ejecutas una creación ordinaria de índice:
```sql
-- ❌ PELIGRO: Bloquea escrituras en producción
CREATE INDEX idx_usuarios_email ON usuarios(email);
```
El motor bloqueará por completo las escrituras (`INSERT`/`UPDATE`) en la tabla `usuarios` mientras lee y construye el árbol B-Tree en disco, congelando tu API de producción durante minutos si la tabla tiene millones de filas.

Para resolver esto, los ingenieros de sistemas relacionales emplean la cláusula **`CONCURRENTLY`**:
```sql
-- ✅ SEGURO: Zero-Downtime
CREATE INDEX CONCURRENTLY idx_usuarios_email ON usuarios(email);
```

### ¿Cómo opera `CONCURRENTLY` bajo el capó?
PostgreSQL realiza una **estrategia de doble escaneo físico de disco**:
1. **Paso 1 (Primer Escaneo)**: PostgreSQL recorre la tabla de fondo y construye una versión preliminar del índice en memoria, permitiendo que los usuarios sigan insertando y modificando filas de forma normal.
2. **Paso 2 (Segundo Escaneo)**: Recorre de nuevo el sistema buscando transacciones concurrentes pendientes para inyectarlas y sincronizarlas en el índice final.
* **El Trade-off**: Tarda el doble de tiempo en crearse y consume más procesador temporalmente, pero **nunca bloquea la base de datos en producción**, garantizando alta disponibilidad total.

---

## 2.5 Implementación en TypeScript de Migración de Índices

Implementemos un script seguro de base de datos en TypeScript para inicializar índices compuestos y GIN utilizando queries parametrizadas robustas:

#### [indexMigration.ts](file:///Users/andres/Documents/biblioteca/sql-databases-programming-book/src/migrations/indexMigration.ts)
```typescript
import { dbPool } from '../clients/dbClient';

export async function ejecutarMigracionesDeIndices(): Promise<void> {
  const cliente = await dbPool.connect();

  try {
    console.log('[Migración] Iniciando creación de índices concurrentes...');

    // 1. Crear un índice B-Tree compuesto siguiendo la regla ESR de forma concurrente
    //  PostgreSQL exige ejecutar CONCURRENTLY fuera de bloques transaccionales explícitos
    await cliente.query(`
      CREATE INDEX CONCURRENTLY IF NOT EXISTS idx_usuarios_activo_registro 
      ON usuarios(activo, fecha_registro DESC);
    `);
    console.log('[Migración] Índice B-Tree compuesto (activo, fecha_registro) creado.');

    // 2. Crear un índice GIN para búsquedas ultra-rápidas dentro de JSONB
    //  El operador 'jsonb_path_ops' optimiza el espacio físico en disco
    await cliente.query(`
      CREATE INDEX CONCURRENTLY IF NOT EXISTS idx_usuarios_preferencias_gin 
      ON usuarios USING gin (preferencias jsonb_path_ops);
    `);
    console.log('[Migración] Índice GIN sobre columna JSONB "preferencias" creado con éxito.');

  } catch (error) {
    console.error('[Migración-Error] Fallo al crear los índices en caliente:', error);
    throw error;
  } finally {
    // Liberamos el cliente de vuelta al Pool de conexiones
    cliente.release();
  }
}
```

---

## Resumen del Capítulo

* Los índices aceleran lecturas reduciendo el I/O físico de disco, pero penalizan la velocidad de escrituras al obligar al motor a mantener los árboles actualizados.
* Un **B-Tree** es un árbol balanceado auto-organizado que garantiza tiempos de acceso logarítmicos $O(\log N)$ para búsquedas exactas, parciales u ordenamientos.
* Los **índices GIN** operan como índices de conceptos invertidos, descomponiendo arrays o JSONB en claves atómicas apuntando a listas de IDs físicos de disco.
* La cláusula **`CONCURRENTLY`** es obligatoria en entornos de alta concurrencia para crear índices en disco mediante doble escaneo secuencial sin bloquear escrituras concurrentes.

En el próximo capítulo, desmitificaremos el núcleo de la confiabilidad relacional: las **Transacciones y Garantías ACID**, profundizando en el Write-Ahead Logging (WAL) y los Checkpoint threads.

---

[← Capítulo anterior (Capítulo 1)](01-fundamentos-y-arquitectura.md) | [Inicio (README.md)](README.md) | [Capítulo siguiente (Capítulo 3) →](03-transacciones-y-acid.md)
