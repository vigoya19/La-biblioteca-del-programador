# Capítulo 1: Fundamentos del Modelo Relacional e Internals del Motor

> "SQL no es un lenguaje imperativo donde ordenas a la máquina CÓMO hacer las cosas; es un lenguaje declarativo matemático donde describes QUÉ resultado deseas y dejas que la física interna del motor calcule la ruta de ejecución óptima."

Durante más de cinco décadas, a pesar de los picos de moda de las tecnologías no relacionales, las bases de datos relacionales tradicionales basadas en el modelo formalizado por Edgar F. Codd en 1970 han permanecido como el núcleo inquebrantable de la informática corporativa. 

Para escribir bases de datos a escala de alto rendimiento, no basta con saber memorizar comandos `SELECT` o `JOIN`. Es fundamental entender la física y la ingeniería mecánica interna que se dispara dentro del motor de base de datos (con **PostgreSQL** como el estándar indiscutible de producción) desde que una cadena de texto SQL entra por el socket de red hasta que los bytes son extraídos de los bloques físicos del disco duro.

---

## 1.1 El Enfoque Declarativo frente al Imperativo

En la programación tradicional (como TypeScript, Go o Java), programamos de forma **Imperativa**: le ordenamos detalladamente a la CPU el camino paso a paso que debe seguir (bucles `for`, punteros, condicionales `if`) para filtrar y ordenar un array de datos.

En **SQL**, el paradigma es **Declarativo**:
* **Tú describes el resultado que deseas**: *"Quiero el email de los 10 usuarios de España con más compras"* y nunca el algoritmo lógico para buscarlo.
* **El Motor calcula el cómo**: Un motor relacional cuenta con un optimizador inteligente basado en modelos de costo matemático que analiza las estadísticas de disco y decide de forma dinámica el mejor algoritmo físico para resolver tu petición.

---

## 1.2 Anatomía del Motor: La Ruta de Ejecución de una Query

Cuando envías una consulta SQL a PostgreSQL a través de un socket TCP, la petición recorre una cadena de montaje de software altamente optimizada dividida en cuatro etapas críticas:

```
    [ Consulta SQL en Texto Plano ]
                  │
                  ▼
         ┌──────────────────┐
         │ 1. Parser/Lexer  │ ◄── [ Valida sintaxis y crea el Árbol AST ]
         └────────┬─────────┘
                  │
                  ▼
         ┌──────────────────┐
         │  2. Analyzer     │ ◄── [ Valida esquemas, tablas, campos y permisos ]
         └────────┬─────────┘
                  │
                  ▼
         ┌──────────────────┐
         │   3. Planner &   │ ◄── [ Diseña planes físicos alternativos ]
         │    Optimizer     │ ◄── [ Selecciona la ruta de menor costo físico ]
         └────────┬─────────┘
                  │
                  ▼
         ┌──────────────────┐
         │   4. Executor    │ ◄── [ Interactúa con Shared Buffers y lee el disco ]
         └────────┬─────────┘
                  │
                  ▼
         [ Recordset de Salida ]
```

### 1. Parser & Lexer (El Validador Ortográfico)
Toma la cadena de texto plano y la descompone en tokens lógicos (palabras clave, identificadores de tablas, operadores). 
* **AST (Abstract Syntax Tree)**: Genera una estructura de árbol en memoria que representa la sintaxis de la consulta. Si cometes un error escribiendo `SELECTT`, esta etapa detiene la ejecución inmediatamente arrojando un error de sintaxis.

### 2. Analyzer & Rewriter (El Validador Semántico y Reglas)
El Parser no sabe si la tabla `usuarios` existe realmente o si tienes permisos para leerla; solo sabe que está sintácticamente bien escrita. El **Analyzer** toma el árbol AST y:
* **Consulta el Catálogo del Sistema**: Verifica esquemas, tipos y privilegios accediendo a las tablas de metadatos más importantes de PostgreSQL:
  * **`pg_class`**: Almacena la relación física de tablas, índices, secuencias y vistas (cada entrada tiene un identificador único en disco llamado `oid` y un puntero físico al archivo principal llamado `relfilenode`).
  * **`pg_attribute`**: Contiene la definición microscópica de cada columna (su tipo de datos, longitud en bytes, nulidad y orden físico en la fila).
  * **`pg_type`**: Describe todos los tipos de datos nativos y personalizados del sistema.
* **El Rewriter**: Reescribe la consulta si es necesario (por ejemplo, expande las Vistas lógicas en sus consultas subyacentes originales o aplica políticas de seguridad).

---

## 1.3 La Física del Almacenamiento en Disco: Páginas de 8KB

Para comprender cómo opera el Executor, debemos entender cómo almacena PostgreSQL la información en caliente en el disco duro. En PostgreSQL, **todos los datos de una tabla se organizan físicamente en archivos segmentados en bloques o páginas de exactamente 8KB (8192 bytes)**.

Cada página de 8KB tiene una estructura interna dividida milimétricamente para maximizar la velocidad de lectura secuencial y aleatoria:

```
┌────────────────────────────────────────────────────────┐
│ PageHeaderData (24 bytes)                              │
│ (Metadatos: pd_lsn, pd_checksum, pd_lower, pd_upper)   │
├────────────────────────────────────────────────────────┤
│ Line Pointers / ItemIdData (4 bytes c/u)               │
│ [ Item 1 ] ───► [ Item 2 ] ───►                        │
├────────────────────────────────────────────────────────┤
│                    ◄─── Free Space ───►                │
├────────────────────────────────────────────────────────┤
│                                 ◄─── [ Tupla Física 2 ]│
├────────────────────────────────────────────────────────┤
│                                 ◄─── [ Tupla Física 1 ]│
└────────────────────────────────────────────────────────┘
```

1. **`PageHeaderData` (24 bytes)**: Ubicado al inicio de la página. Almacena metadatos del bloque, incluyendo:
   * `pd_lsn`: El Log Sequence Number de la última transacción WAL que modificó esta página (crucial para recovery).
   * `pd_lower`: Puntero al inicio del espacio libre de la página (crece hacia abajo a medida que añadimos índices de fila).
   * `pd_upper`: Puntero al inicio de la primera tupla física de datos (crece hacia arriba a medida que añadimos registros).
2. **`ItemIdData` / Line Pointers (4 bytes cada uno)**: Un array de punteros ligeros que crecen de izquierda a derecha. Contienen el offset en bytes exacto donde comienza la tupla física en el bloque y su tamaño.
3. **Free Space (Espacio Libre)**: Zona vacía en medio del bloque que se reduce dinámicamente con las inserciones.
4. **Heap Tuples / Tuplas Físicas**: Los registros reales que se insertan de derecha a izquierda (de abajo hacia arriba). Cada tupla inicia con una cabecera de metadatos de fila (**`HeapTupleHeaderData`** de 23 bytes) que contiene metadatos clave como visibilidad MVCC (`xmin`, `xmax`) y alineación de datos físicos.

---

## 1.4 El Cost-Based Optimizer (CBO) y sus Matemáticas de Coste

El **Planner & Optimizer** calcula el costo teórico de cada plan alternativo de acceso a disco basándose en unidades de coste de CPU e I/O. La unidad básica de coste (`1.0`) equivale al costo estimado de leer secuencialmente una sola página de 8KB desde el disco.

### Las Variables Fundamentales del Catálogo:
Estas variables son configurables en el archivo `postgresql.conf` para calibrar el motor de acuerdo al hardware (por ejemplo, ajustándolas a la baja si usamos discos de estado sólido NVMe ultrarrápidos):
* **`seq_page_cost`** (Por defecto = `1.0`): Coste de leer secuencialmente un bloque de 8KB.
* **`random_page_cost`** (Por defecto = `4.0`): Coste de leer aleatoriamente un bloque de 8KB (mayor coste debido a saltos de cabezal en discos mecánicos tradicionales).
* **`cpu_tuple_cost`** (Por defecto = `0.01`): Coste de CPU para procesar una sola tupla.
* **`cpu_index_tuple_cost`** (Por defecto = `0.005`): Coste de CPU para procesar una fila dentro de un índice.
* **`cpu_operator_cost`** (Por defecto = `0.0025`): Coste de procesar un operador lógico (ej. evaluar un filtro `edad > 30`).

### La Fórmula Matemática de un Escaneo Secuencial (Seq Scan):

El planificador calcula el coste total de leer una tabla completa mediante un Seq Scan aplicando la siguiente ecuación:

$$\text{Costo Total} = (N_{\text{páginas}} \times \text{seq\_page\_cost}) + (N_{\text{tuplas}} \times \text{cpu\_tuple\_cost}) + (N_{\text{tuplas\_filtradas}} \times \text{cpu\_operator\_cost})$$

### Ejemplo Matemático Resuelto Paso a Paso:

Imagina que tenemos una tabla `usuarios` en producción con las siguientes métricas físicas guardadas en el catálogo de estadísticas:
* La tabla ocupa **$1,500$ páginas físicas** en el disco duro ($N_{\text{páginas}} = 1500$).
* Alberga un total de **$100,000$ tuplas** ($N_{\text{tuplas}} = 100000$).
* Queremos ejecutar la consulta: `SELECT * FROM usuarios WHERE activo = true;`
* El optimizador estima (basado en histogramas de estadísticas) que procesará las $100,000$ filas para evaluar el filtro.

Calculamos el costo utilizando los valores por defecto de PostgreSQL:
1. **Coste de lectura física de I/O**:
   $$1,500 \text{ páginas} \times 1.0 \text{ (seq\_page\_cost)} = 1500.0$$
2. **Coste de procesamiento de CPU de tuplas**:
   $$100,000 \text{ tuplas} \times 0.01 \text{ (cpu\_tuple\_cost)} = 1000.0$$
3. **Coste de evaluación del operador lógico en CPU**:
   $$100,000 \text{ evaluaciones} \times 0.0025 \text{ (cpu\_operator\_cost)} = 250.0$$
4. **Costo Total Estimado del Plan**:
   $$\text{Costo Total} = 1500.0 + 1000.0 + 250.0 = 2750.0$$

El planificador comparará este coste de `2750.0` frente al costo estimado de usar un índice (que implicará lecturas aleatorias de disco `random_page_cost` pero sobre una fracción minúscula de páginas) y seleccionará el plan de menor número de unidades.

---

### 3. Planner & Optimizer (El Director de Orquesta)
El cerebro matemático absoluto del motor. Su misión es tomar el árbol semántico y diseñar múltiples **Planes de Ejecución** alternativos viables para obtener tus datos.
* **Cost-Based Optimizer (CBO)**: El optimizador calcula el "Costo Matemático" (representado en unidades de acceso a páginas de disco) de cada plan alternativo.
* **Análisis de Estadísticas**: PostgreSQL mantiene estadísticas periódicas sobre tus tablas (porcentaje de valores nulos, distribución de valores, cantidad de filas en `pg_statistic`). El optimizador las consulta para decidir: *¿Es más rápido recorrer la tabla completa secuencialmente (Seq Scan) o buscar en este índice B-Tree (Index Scan)? ¿Conviene unir estas tablas mediante Hash Join o Merge Join?*
* El plan de menor costo estimado se empaqueta y se entrega al Executor.

### 4. Executor (Los Actores)
El Executor recibe el plan físico óptimo y procesa los datos reales.
* **Shared Buffers (Memoria RAM)**: Antes de tocar el disco físico, el Executor consulta los búferes de memoria compartida de PostgreSQL. Si las páginas de datos buscadas ya residen en la RAM, las devuelve instantáneamente, evitando latencias físicas de lectura.
* **Disco Duro**: Si los datos no están en memoria, interactúa con el sistema de archivos del sistema operativo para cargar las páginas de disco duro a la RAM y devolver el recordset final.

> [!NOTE]
> ### 🎭 El Guionista, el Inspector, el Director de Teatro y el Elenco de Actores
> 
> Visualicemos el viaje de una consulta SQL dentro de PostgreSQL:
> 
> - **La Query (El Guión)**: El programador escribe un guión teatral: *"Hacer que Julieta y Romeo se encuentren en el balcón del palacio"*.
> - **El Parser (Inspector Ortográfico)**: Toma el papel y revisa que no haya faltas de ortografía. Si escribiste *"Julietta"* o te saltaste un punto y coma, detiene la obra y grita: *"¡Error en la línea 1!"*.
> - **El Analyzer (El Agente de Casting)**: Revisa el catálogo de actores del teatro (las tablas). Verifica que *"Romeo"* y *"Julieta"* estén contratados en el elenco del teatro y que tengas los derechos de autor (los permisos de usuario `GRANT`) para hacer la representación de la obra.
> - **El Optimizer/Planner (El Director de la Obra)**: Es el estratega. Toma el guión y diseña **3 formas físicas distintas de filmar la escena**:
>   - *Plan A*: Romeo sube por las escaleras principales del palacio (Index Scan).
>   - *Plan B*: Romeo escala por una cuerda directamente al balcón (Index Only Scan).
>   - *Plan C*: Romeo recorre secuencialmente todas las habitaciones del palacio abriendo puerta por puerta buscando a Julieta (COLLSCAN / Seq Scan lento).
>   - El Director consulta los metadatos: *"Romeo pesa 80 kilos y la cuerda resiste 100 kilos. Subir por la cuerda tarda 5 segundos. Recorrer todas las habitaciones de la mansión tarda 2 horas"*. El Director descarta el Plan C por ser estúpido y costoso, y elige el **Plan B** por tener el menor costo físico de energía y tiempo.
> - **El Executor (Los Actores)**: Los actores reales salen al escenario físico (el disco duro y la memoria RAM compartida), corren, colocan la escalera, escalan y actúan bajo la dirección del Plan B elegido. El espectador solo ve la obra terminada con éxito.

---

## 1.3 Conexión e Implementación Segura en TypeScript

Para interactuar con PostgreSQL de forma profesional y prevenir inyecciones SQL a nivel de driver en Node.js/TypeScript, empleamos el controlador líder `pg` y aserciones estrictas:

#### [dbClient.ts](file:///Users/andres/Documents/biblioteca/sql-databases-programming-book/src/clients/dbClient.ts)
```typescript
import { Pool } from 'pg';

// 1. Inicializar un Connection Pool (Grupo de conexiones reutilizables)
export const dbPool = new Pool({
  host: '127.0.0.1',
  port: 5432,
  user: 'postgres_user',
  password: 'super_secure_db_pass_789',
  database: 'biblioteca_relacional',
  max: 20, // Máximo de conexiones concurrentes en el pool
  idleTimeoutMillis: 30000,
  connectionTimeoutMillis: 2000
});
```

#### [usuarioService.ts](file:///Users/andres/Documents/biblioteca/sql-databases-programming-book/src/services/usuarioService.ts)
```typescript
import { dbPool } from '../clients/dbClient';

export interface UsuarioRelacional {
  id: number;
  nombre: string;
  email: string;
  activo: boolean;
}

// 2. Buscar usuarios de forma segura utilizando queries parametrizadas
export async function buscarUsuarioPorEmail(email: string): Promise<UsuarioRelacional | null> {
  // ❌ NUNCA concatenes strings directamente: `WHERE email = '${email}'` (SQL Injection catastrófico)
  //  Utilizamos marcadores de parámetros '$1' para que el driver sanitice la entrada
  const queryText = `
    SELECT id, nombre, email, activo 
    FROM usuarios 
    WHERE email = $1
    LIMIT 1
  `;
  
  const values = [email];

  // El Pool gestiona el préstamo y retorno del socket de conexión de forma automática
  const resultado = await dbPool.query(queryText, values);

  if (resultado.rows.length === 0) {
    return null;
  }

  // Retornar el registro fuertemente tipado
  return {
    id: resultado.rows[0].id,
    nombre: resultado.rows[0].nombre,
    email: resultado.rows[0].email,
    activo: resultado.rows[0].activo
  };
}
```

---

## Resumen del Capítulo

* SQL opera bajo el **paradigma declarativo**, donde el desarrollador describe el resultado deseado y el motor se encarga de calcular el algoritmo de acceso óptimo.
* El viaje de una consulta SQL dentro de PostgreSQL atraviesa de forma secuencial el **Parser** (sintaxis AST), el **Analyzer** (esquemas y permisos), el **Planner/Optimizer** (costo estimado CBO) y el **Executor** (Shared Buffers y lectura física de disco).
* El **Connection Pool** optimiza la latencia de las conexiones reutilizando sockets de red abiertos en caliente para evitar el costo de handshake de TCP en cada petición.
* Las **queries parametrizadas** son la primera y más robusta línea de defensa contra ataques de inyección SQL, abstrayendo los valores lógicos de la compilación de la consulta.

En el próximo capítulo, estudiaremos el cimiento físico de la velocidad relacional: la **Anatomía de la Indexación**, analizando de forma milimétrica los internals de los árboles **B-Tree**, índices **Hash** e índices invertidos **GIN** para datos JSONB.

---

[Inicio (README.md)](README.md) | [Capítulo siguiente (Capítulo 2) →](02-anatomia-indexacion.md)
