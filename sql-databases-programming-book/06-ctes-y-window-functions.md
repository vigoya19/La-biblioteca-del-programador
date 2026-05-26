# Capítulo 6: Consultas Avanzadas: CTEs y Window Functions

> "Las bases de datos relacionales no son simples cajones de almacenamiento; son potentes motores de álgebra relacional de conjuntos. Si estás procesando bucles recursivos o cálculos de agregación secuencial en tu backend de Node.js en lugar de usar CTEs y Window Functions en SQL, estás ahogando tu infraestructura en latencia de red."

A medida que las aplicaciones escalan, surge la necesidad de realizar análisis avanzados directamente en la base de datos: obtener clasificaciones de ventas por categoría, analizar diferencias de tiempo secuenciales en el comportamiento del usuario, u obtener organigramas corporativos jerárquicos recursivos. 

Históricamente, los programadores solucionaban esto haciendo múltiples consultas consecutivas en bucle (`N+1`) o descargando millones de registros a la memoria RAM de su servidor de aplicaciones para procesarlos iterativamente. En este capítulo, aprenderemos cómo resolver estos problemas de manera profesional y en un solo viaje de red (`round-trip`) utilizando **Common Table Expressions (CTEs)** simples y recursivas, y las increíbles **Window Functions (Funciones de Ventana)** de PostgreSQL.

---

## 6.1 Common Table Expressions (CTEs)

Una **CTE (Common Table Expression)** permite definir un conjunto de resultados temporal con nombre que existe exclusivamente dentro del alcance de una sola consulta (`SELECT`, `INSERT`, `UPDATE` o `DELETE`). Es la forma relacional de definir variables reutilizables o subconsultas sumamente legibles.

### 1. Ventajas Clave sobre Subconsultas:
* **Legibilidad impecable**: Te permite estructurar consultas secuenciales de arriba a abajo, evitando el "síndrome de la cebolla" (múltiples subconsultas anidadas difíciles de mantener).
* **Modularidad**: Puedes reutilizar la misma CTE múltiples veces en diferentes uniones (`JOIN`) dentro de la consulta principal.

### 2. Sintaxis Básica de una CTE:
```sql
WITH ventas_usuario AS (
  SELECT usuario_id, SUM(monto) AS total_gastado
  FROM pedidos
  GROUP BY usuario_id
)
SELECT u.nombre, v.total_gastado
FROM usuarios u
JOIN ventas_usuario v ON u.id = v.usuario_id
WHERE v.total_gastado > 1000;
```

---

## 6.2 CTEs Recursivas: Navegando Estructuras de Datos Complejas

Uno de los mayores poderes de SQL es su capacidad de resolver algoritmos de grafos o jerarquías complejas sin límites de profundidad mediante el modificador **`WITH RECURSIVE`**. 

Una CTE recursiva consta de tres partes fundamentales que se ejecutan cíclicamente:
1. **Miembro Ancla (Anchor Member)**: La consulta base inicial que arranca el proceso (no recursiva).
2. **UNION o UNION ALL**: La costura que une los resultados previos con el siguiente paso recursivo.
3. **Miembro Recursivo (Recursive Member)**: La consulta que se une a sí misma, haciendo referencia al nombre de la propia CTE y filtrando la condición de parada mediante el `JOIN`.

```sql
WITH RECURSIVE organigrama AS (
  -- 1. Miembro Ancla: Seleccionamos al CEO (quien no tiene jefe)
  SELECT id, nombre, jefe_id, 1 AS nivel
  FROM empleados
  WHERE jefe_id IS NULL
  
  UNION ALL
  
  -- 2. Miembro Recursivo: Unimos a los subordinados inmediatos de la iteración previa
  SELECT e.id, e.nombre, e.jefe_id, o.nivel + 1
  FROM empleados e
  JOIN organigrama o ON e.jefe_id = o.id
)
SELECT * FROM organigrama;
```

---

## 6.3 Window Functions (Funciones de Ventana)

Las **Window Functions** ejecutan un cálculo de agregación sobre un conjunto de filas de la tabla que están directamente relacionadas con la fila actual. 

A diferencia de un `GROUP BY` convencional (que fusiona y reduce tus registros en una única fila agrupada colapsando el detalle), **las Window Functions no reducen las filas devueltas**. Mantienen cada registro individual intacto en la salida, anexando el resultado del cálculo agrupado como una columna adicional de metadatos.

### Cláusulas Fundamentales:
* **`OVER()`**: Indica que la función debe ejecutarse como una función de ventana.
* **`PARTITION BY`**: Define las fronteras de los grupos (equivalente al "agrupar por" pero interno para el cálculo de la fila).
* **`ORDER BY`**: Define el orden de procesamiento secuencial de las filas dentro de cada partición (crítico para clasificaciones o cálculos acumulados).

---

> [!NOTE]
> ### 🪟 La Ventana Deslizante del Autobús y el Espejo Retrovisor
> 
> Visualicemos las Window Functions y las CTEs Recursivas a través de analogías físicas cotidianas:
> 
> - **El Efecto de las Window Functions (La Ventana Deslizante de un Autobús)**:
>   - Imagina que vas viajando sentado en un autobús de pasajeros turístico. 
>   - A medida que avanzas por la carretera, miras hacia el exterior a través de un panel de vidrio rectangular deslizable (**La Ventana `OVER`**).
>   - A través de esa ventana, solo puedes ver a un grupo específico de pasajeros que camina por la calle junto a ti en un momento dado (**La Partición `PARTITION BY`**).
>   - Tú sigues siendo un pasajero individual con tu propio asiento y boleto intactos (las filas no se colapsan). 
>   - Pero puedes hacer un cálculo dinámico en tiempo real sobre las personas que están visibles en tu marco de visión: por ejemplo, contar cuántos llevan camiseta roja a medida que el autobús avanza, o calcular la estatura promedio de tu grupo inmediato. Tu ventana se despliza fila por fila, recalculando el valor de forma aislada y dinámica.
> 
> - **El Cálculo Secuencial (`LEAD` y `LAG` - El Espejo Retrovisor y el Parabrisas)**:
>   - Imagina que estás conduciendo tu coche en una autopista de un solo carril de forma ordenada (`ORDER BY`).
>   - Quieres saber la velocidad del coche que va justo detrás de ti para ajustar tu distancia de frenado.
>   - En lugar de detener el tráfico y bajarte a preguntar (`N+1`), miras por tu espejo retrovisor (**Función `LAG`**). Tienes acceso inmediato a los datos de la fila anterior en la autopista sin salirte de tu propio coche.
>   - Si quieres ver la velocidad del coche de adelante, miras a través de tu parabrisas (**Función `LEAD`**). La base de datos calcula instantáneamente el desfase en una sola pasada lógica.
> 
> - **La CTE Recursiva (La Reacción en Cadena de Dominós)**:
>   - Imagina que tienes una fila serpenteante de miles de piezas de dominó alineadas una tras otra.
>   - No necesitas empujar cada dominó manualmente uno por uno (**procesamiento iterativo en Node.js**).
>   - Simplemente empujas la primera pieza (**El Miembro Ancla**). Esa caída golpea de forma natural y física a las piezas subsecuentes (**El Miembro Recursivo** mediante `JOIN`), propagándose de forma fluida a lo largo de toda la cadena hasta agotar las piezas (**La Condición de Parada**).

---

## 6.4 Implementación Práctica en TypeScript

A continuación, implementaremos un servicio de análisis corporativo avanzado en TypeScript que utiliza PostgreSQL nativo para calcular:
1. Un organigrama jerárquico recursivo de empleados completo con su nivel de profundidad.
2. Un cálculo analítico de ventas acumulado mensual y comparativo con el mes anterior (`LAG`) de forma instantánea.

### `analisisMétricas.ts`
```typescript
import { dbPool } from '../clients/dbClient';

export interface JerarquiaEmpleado {
  id: number;
  nombre: string;
  puesto: string;
  jefeId: number | null;
  nivel: number;
  rutaJerarquia: string;
}

export interface ReporteFinancieroAnalitico {
  mesAnio: string;
  ventasMensuales: number;
  ventasAcumuladasHistoricas: number;
  ventasMesAnterior: number | null;
  porcentajeVariacion: number | null;
}

// 1. Obtener jerarquía recursiva completa a partir de un empleado raíz (ej: un Gerente o el CEO)
export async function obtenerArbolJerarquico(empleadoRaizId: number): Promise<JerarquiaEmpleado[]> {
  const cliente = await dbPool.connect();

  try {
    const sqlJerarquia = `
      WITH RECURSIVE arbol_empleados AS (
        -- Miembro Ancla: El empleado de origen seleccionado
        SELECT 
          id, 
          nombre, 
          puesto, 
          jefe_id, 
          1 AS nivel,
          nombre::text AS ruta_jerarquia
        FROM empleados
        WHERE id = $1

        UNION ALL

        -- Miembro Recursivo: Unimos recursivamente a los subordinados directos
        SELECT 
          e.id, 
          e.nombre, 
          e.puesto, 
          e.jefe_id, 
          ae.nivel + 1 AS nivel,
          (ae.ruta_jerarquia || ' -> ' || e.nombre)::text AS ruta_jerarquia
        FROM empleados e
        JOIN arbol_empleados ae ON e.jefe_id = ae.id
      )
      SELECT id, nombre, puesto, jefe_id AS "jefeId", nivel, ruta_jerarquia AS "rutaJerarquia"
      FROM arbol_empleados
      ORDER BY nivel, nombre;
    `;

    const resultado = await cliente.query(sqlJerarquia, [empleadoRaizId]);
    return resultado.rows;

  } catch (error: any) {
    console.error('Error al generar árbol jerárquico recursivo:', error.message);
    throw error;
  } finally {
    cliente.release();
  }
}

// 2. Generar reporte analítico con Window Functions (Cálculo acumulado e incremento intermensual)
export async function generarReporteMensualVentas(): Promise<ReporteFinancieroAnalitico[]> {
  const cliente = await dbPool.connect();

  try {
    const sqlAnalisis = `
      WITH ventas_mensuales_base AS (
        SELECT 
          TO_CHAR(fecha, 'YYYY-MM') AS mes,
          SUM(total) AS monto_total
        FROM facturas
        GROUP BY TO_CHAR(fecha, 'YYYY-MM')
      )
      SELECT 
        mes AS "mesAnio",
        monto_total AS "ventasMensuales",
        -- 1. SUM acumulativo sobre la ventana temporal ordenada históricamente
        SUM(monto_total) OVER (
          ORDER BY mes ASC
          ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
        ) AS "ventasAcumuladasHistoricas",
        -- 2. LAG para obtener el valor exacto del mes cronológico anterior
        LAG(monto_total, 1, NULL) OVER (
          ORDER BY mes ASC
        ) AS "ventasMesAnterior"
      FROM ventas_mensuales_base
      ORDER BY mes ASC;
    `;

    const resultado = await cliente.query(sqlAnalisis);
    
    return resultado.rows.map(row => {
      const ventasMensuales = parseFloat(row.ventasMensuales);
      const ventasMesAnterior = row.ventasMesAnterior ? parseFloat(row.ventasMesAnterior) : null;
      let porcentajeVariacion: number | null = null;

      if (ventasMesAnterior && ventasMesAnterior > 0) {
        porcentajeVariacion = parseFloat(((ventasMensuales - ventasMesAnterior) / ventasMesAnterior * 100).toFixed(2));
      }

      return {
        mesAnio: row.mesAnio,
        ventasMensuales,
        ventasAcumuladasHistoricas: parseFloat(row.ventasAcumuladasHistoricas),
        ventasMesAnterior,
        porcentajeVariacion
      };
    });

  } catch (error: any) {
    console.error('Error al generar reporte de ventas analíticas:', error.message);
    throw error;
  } finally {
    cliente.release();
  }
}
```

---

## Resumen del Capítulo

* Las **CTEs (`WITH`)** encapsulan lógica temporal mejorando la estructura, legibilidad y reutilización de fragmentos de consulta de forma limpia y organizada de arriba a abajo.
* Las **CTEs Recursivas (`WITH RECURSIVE`)** resuelven de forma eficiente la navegación y consulta de jerarquías de árbol ilimitadas u operaciones de grafos directamente en la base de datos.
* Las **Window Functions** operan sobre una ventana de registros relacionados (`OVER`) sin colapsar la salida del resultado, preservando los detalles individuales del registro original.
* Cláusulas como **`PARTITION BY`** aíslan conjuntos de datos para cálculos selectivos, mientras que funciones avanzadas como **`LAG`** y **`LEAD`** permiten acceder a filas adyacentes de manera secuencial sin duplicar viajes al motor.

En el próximo capítulo, profundizaremos en el diseño de bases de datos de alto nivel mediante el estudio de **Normalización Avanzada, Desnormalización y la batalla relacional/documental: EAV vs. JSONB**.

---

[← Capítulo anterior (Capítulo 5)](05-bloqueos-locking.md) | [Inicio (README.md)](README.md) | [Capítulo siguiente (Capítulo 7) →](07-modelado-y-normalizacion.md)
