# Capítulo 9: Particionamiento y Sharding de Tablas

> "Cuando una sola tabla de tu base de datos supera los 100 millones de filas, los índices B-Tree dejan de caber en la memoria RAM y cada búsqueda física en disco se vuelve una pesadilla de latencia. En ese momento, tu única alternativa de supervivencia es dividir el problema físico en fragmentos manejables."

En los capítulos de indexación y optimización estudiamos cómo acelerar consultas individuales. Sin embargo, a medida que un sistema de software tiene éxito, las tablas históricas (como transacciones bancarias, logs de auditoría o eventos IoT) crecen a un ritmo vertiginoso de gigabytes diarios. 

Cuando una tabla relacional alcanza magnitudes gigantescas, comandos ordinarios como `VACUUM`, `CREATE INDEX` o reconstrucciones de tablas se vuelven imposibles de ejecutar en caliente debido al tiempo y al bloqueo masivo. En este capítulo, aprenderemos a dominar la división de datos a gran escala mediante el **Particionamiento Nativo de PostgreSQL** en un solo nodo, comprenderemos el poder del **Partition Pruning (Poda de Particiones)** y analizaremos conceptualmente el escalamiento horizontal multihilo mediante **Sharding Relacional**.

---

## 9.1 Particionamiento Nativo en PostgreSQL (Single-Node Scaling)

El particionamiento consiste en dividir lógicamente una tabla gigante (llamada **Tabla Padre**) en múltiples tablas físicas más pequeñas e independientes (llamadas **Tablas Particiones**), de forma totalmente transparente para la capa de la aplicación. Tu backend sigue haciendo `SELECT` o `INSERT` contra la Tabla Padre, y el motor relacional se encarga de rutear físicamente los datos al fragmento correcto bajo el capó.

PostgreSQL soporta tres estrategias nativas de particionamiento declarativo:

### 1. Particionamiento por Rango (Range Partitioning)
* **Cómo funciona**: Los datos se dividen según rangos contiguos definidos sobre una o varias columnas clave.
* **Caso de uso por excelencia**: Tablas históricas de eventos o facturación segmentadas de forma natural por meses o años (ej: `fecha`).

### 2. Particionamiento por Lista (List Partitioning)
* **Cómo funciona**: Los datos se asignan explícitamente a particiones que corresponden a valores puntuales definidos en una lista de categorías.
* **Caso de uso**: Segmentación por regiones geográficas (`pais IN ('MX', 'CO', 'AR')`) o estados de negocio (`estado IN ('activo', 'cancelado')`).

### 3. Particionamiento por Hash (Hash Partitioning)
* **Cómo funciona**: El motor aplica una función hash criptográfica sobre el campo clave y divide los registros de forma uniformemente balanceada entre un número predefinido de particiones físicas.
* **Caso de uso**: Ideal cuando no hay un rango o lista natural de segmentación y queremos distribuir uniformemente cargas masivas de escrituras concurrentes.

---

## 9.2 El Superpoder Físico: Partition Pruning (Poda de Particiones)

El verdadero beneficio en rendimiento del particionamiento ocurre gracias a una técnica automática del planificador llamada **Partition Pruning (Poda de Particiones)**.

Si tienes una tabla particionada mensualmente por `fecha` a lo largo de 5 años (60 particiones físicas en disco) y ejecutas la consulta:
```sql
SELECT * FROM metricas_historicas WHERE fecha BETWEEN '2026-05-01' AND '2026-05-31';
```
El optimizador físico descarta inmediatamente 59 de las 60 particiones antes de tocar el disco. Va **únicamente** a escanear los archivos de datos de la partición de mayo de 2026.
* **Resultado**: Reducción drástica del espacio de búsqueda. Los índices B-Tree requeridos para resolver la consulta son minúsculos y caben por completo en la memoria caché compartida RAM (`Shared Buffers`), logrando búsquedas en microsegundos como si la tabla fuera enana.

---

> [!NOTE]
> ### 📂 El Archivador por Carpetas Mensuales y los Edificios de Sucursales
> 
> Visualicemos el particionamiento de tablas y el sharding multihilo con analogías físicas sencillas:
> 
> - **El Particionamiento (El Archivador por Carpetas Mensuales)**:
>   - Imagina que eres un contador que trabaja en una oficina con un archivador metálico de un solo cajón gigante.
>   - Cada vez que generas una factura física, la avientas al cajón sin ningún orden. Al cabo de 10 años, tienes un cerro de 1,000,000 de facturas sueltas dentro de ese único cajón.
>   - Si un auditor te pide buscar la factura número 4829 del mes de Mayo de 2024, tendrás que pasar horas escarbando en la montaña de papel (**Seq Scan de una tabla gigante**).
>   - Decides solucionar esto organizando tu archivador. Instalas separadores de carpetas colgantes de colores, uno por cada mes del año (**Particionamiento Declarativo por Rango**).
>   - Cuando el auditor te pide la factura de Mayo de 2024, vas directamente a la carpeta etiquetada *"2024-05"* (**Partition Pruning**), abres únicamente esa carpeta con 50 hojas adentro y extraes la factura en 3 segundos. Descartaste el 99% del archivador físico de forma instantánea con tus ojos antes de tocar el papel.
> 
> - **El Sharding (Los Edificios de Sucursales de Archivo)**:
>   - Tu empresa tiene tanto éxito que el archivador metálico de tu oficina ya no cabe en el edificio. El piso está a punto de colapsar por el peso del papel físico.
>   - Decides que no puedes centralizar el archivo en tu pequeña oficina de la sede central.
>   - Compras tres edificios de almacenamiento secundarios en diferentes ciudades: **Sucursal Norte (Servidor 1)**, **Sucursal Centro (Servidor 2)** y **Sucursal Sur (Servidor 3)**.
>   - Estableces una regla inviolable en la entrada: *"Todos los clientes del Norte mandan sus facturas físicas a la Sucursal Norte, y los del Sur a la Sur"*.
>   - Ahora, el almacenamiento físico de papel está distribuido geográficamente en diferentes cimientos y sistemas operativos (**Sharding de Base de Datos**). Las escrituras de los clientes no compiten por la misma calle ni el mismo cajón, multiplicando la capacidad del negocio hasta el infinito a costa de coordinar camiones de mensajería para unir reportes globales (**Cross-Shard Queries**).

---

## 9.3 Sharding de Base de Datos: Escalabilidad Horizontal

Mientras que el particionamiento divide tus datos dentro de los discos de un **mismo servidor físico** (escalabilidad vertical), el **Sharding** divide y distribuye tus datos a lo largo de **múltiples servidores de base de datos independientes (nodos)** en red.

### Conceptos Clave de Sharding:
* **Shard Key (Clave de Sharding)**: El campo por el cual decidimos segmentar y rutear los datos (ej: `tenant_id` en SaaS o `user_id` en redes sociales). Una buena clave garantiza que los datos de un mismo usuario vivan juntos en el mismo nodo físico, eliminando uniones JOIN costosas en red.
* **Ruteador de Consultas**: Una capa de software (puede ser tu backend de Node.js o proxies especializados como Citus para PostgreSQL) que interpreta hacia qué nodo mandar la consulta basándose en el parámetro del filtro.

---

## 9.4 Implementación en SQL de Particionamiento por Rango y Ruteo en TypeScript

Implementaremos primero un script en código SQL de producción para crear una **Tabla Padre de Facturas** particionada declarativamente de forma nativa por rango de fecha en PostgreSQL, junto con sus particiones mensuales e índices específicos. Posteriormente, escribiremos el código de ruteo y creación dinámica de particiones en TypeScript:

### `crearEsquemaParticionado.sql`
```sql
-- 1. Crear la Tabla Padre utilizando la cláusula PARTITION BY RANGE
CREATE TABLE facturas_historicas (
    id SERIAL,
    usuario_id INT NOT NULL,
    total NUMERIC(12, 2) NOT NULL,
    fecha DATE NOT NULL,
    creado_at TIMESTAMP DEFAULT NOW(),
    -- Importante: La columna de particionamiento DEBE formar parte de la clave primaria compuesta
    PRIMARY KEY (id, fecha)
) PARTITION BY RANGE (fecha);

-- 2. Crear las particiones físicas asociadas a la Tabla Padre
CREATE TABLE facturas_y2026_m04 PARTITION OF facturas_historicas
    FOR VALUES FROM ('2026-04-01') TO ('2026-05-01');

CREATE TABLE facturas_y2026_m05 PARTITION OF facturas_historicas
    FOR VALUES FROM ('2026-05-01') TO ('2026-06-01');

CREATE TABLE facturas_y2026_m06 PARTITION OF facturas_historicas
    FOR VALUES FROM ('2026-06-01') TO ('2026-07-01');

-- 3. Crear índices sobre la Tabla Padre. PostgreSQL creará automáticamente índices B-Tree locales
-- de forma independiente sobre cada partición física de fondo.
CREATE INDEX idx_facturas_fecha_usuario ON facturas_historicas(fecha, usuario_id);
```

### `servicioParticionamiento.ts`
```typescript
import { dbPool } from '../clients/dbClient';

// 1. Guardar factura en la tabla padre (PostgreSQL la rutea automáticamente)
export async function registrarFactura(
  usuarioId: number,
  total: number,
  fechaStr: string // Formato: 'YYYY-MM-DD'
): Promise<boolean> {
  const cliente = await dbPool.connect();

  try {
    // Insertamos directamente en la Tabla Padre
    const sqlInsertar = `
      INSERT INTO facturas_historicas (usuario_id, total, fecha)
      VALUES ($1, $2, $3)
    `;

    await cliente.query(sqlInsertar, [usuarioId, total, fechaStr]);
    return true;

  } catch (error: any) {
    // Si la partición de esa fecha no ha sido creada dinámicamente en disco, PostgreSQL arrojará un error
    if (error.code === '23514') {
      console.warn(`[Particiones] No existe partición física en disco para albergar la fecha ${fechaStr}. Disparando aprovisionamiento dinámico.`);
      
      const exitoCreacion = await aprovisionarParticionParaFecha(fechaStr);
      if (exitoCreacion) {
        // Reintentamos una vez más la inserción tras la migración automática
        return await registrarFactura(usuarioId, total, fechaStr);
      }
    }
    
    console.error('Error al insertar factura en estructura particionada:', error.message);
    return false;
  } finally {
    cliente.release();
  }
}

// 2. Aprovisionar dinámicamente particiones físicas en disco en caliente (Migración automatizada)
export async function aprovisionarParticionParaFecha(fechaStr: string): Promise<boolean> {
  const cliente = await dbPool.connect();
  
  try {
    const fecha = new Date(fechaStr);
    const anio = fecha.getFullYear();
    const mesNum = fecha.getMonth() + 1; // 1-12
    
    const mesStr = mesNum < 10 ? `0${mesNum}` : `${mesNum}`;
    const siguienteMesNum = mesNum === 12 ? 1 : mesNum + 1;
    const siguienteAnio = mesNum === 12 ? anio + 1 : anio;
    const siguienteMesStr = siguienteMesNum < 10 ? `0${siguienteMesNum}` : `${siguienteMesNum}`;

    const nombreParticion = `facturas_y${anio}_m${mesStr}`;
    const rangoInicio = `${anio}-${mesStr}-01`;
    const rangoFin = `${siguienteAnio}-${siguienteMesStr}-01`;

    await cliente.query('BEGIN');

    // Creamos la nueva tabla física de partición unida a la padre
    const sqlCrearParticion = `
      CREATE TABLE IF NOT EXISTS ${nombreParticion} PARTITION OF facturas_historicas
      FOR VALUES FROM ('${rangoInicio}') TO ('${rangoFin}')
    `;
    await cliente.query(sqlCrearParticion);

    await cliente.query('COMMIT');
    console.log(`[Particiones] Partición física '${nombreParticion}' aprovisionada y unida con éxito.`);
    return true;

  } catch (error: any) {
    await cliente.query('ROLLBACK');
    console.error(`[Particiones] Fallo crítico al aprovisionar partición para ${fechaStr}:`, error.message);
    return false;
  } finally {
    cliente.release();
  }
}
```

---

## Resumen del Capítulo

* El **Particionamiento** divide tablas gigantescas de forma transparente en múltiples archivos físicos independientes, evitando cuellos de botella de indexación masiva en tablas de más de 100 millones de registros.
* El **Partition Pruning (Poda)** permite al motor ignorar por completo las particiones físicas fuera del alcance del filtro de la consulta, reduciendo las lecturas de disco al mínimo absoluto de forma automática.
* Las tres estrategias nativas clave de particionamiento declarativo en PostgreSQL son: **Rango**, **Lista** y **Hash**.
* El **Sharding** distribuye la información de la base de datos horizontalmente entre múltiples servidores físicos e independientes en red, escalando escrituras y almacenamiento de forma masiva mediante la adecuada elección de una `Shard Key`.

En el próximo capítulo, estudiaremos los pilares de la consistencia geográfica y escalabilidad de lectura a través de la **Replicación y Alta Disponibilidad** en producción.

---

[← Capítulo anterior (Capítulo 8)](08-explain-analyze-y-joins.md) | [Inicio (README.md)](README.md) | [Capítulo siguiente (Capítulo 10) →](10-replicacion-y-alta-disponibilidad.md)
