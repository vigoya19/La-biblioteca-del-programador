# Capítulo 7: Modelado de Datos y Normalización Avanzada

> "La mala normalización de una base de datos relacional genera redundancia y anomalías de actualización. La sobre-normalización académica destruye el rendimiento del motor forzando decenas de uniones JOIN físicas innecesarias. El arte del modelado avanzado radica en equilibrar la pureza teórica con las realidades físicas del disco y de la CPU."

El diseño de un esquema de base de datos es la base de todo sistema de software escalable. Una estructura mal planteada se arrastra durante años en forma de consultas lentas, parches complejos en el código y riesgos constantes de inconsistencias en caliente.

En este capítulo, desmitificaremos las **Formas Normales** (de la Primera Forma Normal a la rigurosa Forma Normal de Boyce-Codd), entenderemos cuándo y cómo aplicar la **desnormalización estratégica**, y analizaremos en detalle la batalla de modelado de atributos dinámicos: el problemático patrón relacional **EAV (Entity-Attribute-Value)** frente al potente tipo de datos **JSONB estructurado** nativo de PostgreSQL.

---

## 7.1 Formas Normales de Relación (1NF a BCNF)

La normalización es un proceso sistemático para organizar las tablas y minimizar la redundancia de datos. Se basa en una serie de reglas progresivas llamadas **Formas Normales**:

### 1. Primera Forma Normal (1NF): Atomicidad Total
* **Regla**: Cada celda (intersección de fila y columna) debe contener un único valor atómico. No se permiten listas, arreglos o grupos repetitivos en un solo campo. Además, la tabla debe contar con una clave primaria definida.
* **Ejemplo incorrecto**: Columna `telefonos` guardando `"555-1234, 555-5678"` en una sola fila.
* **Solución**: Separar los teléfonos en registros independientes o moverlos a una tabla relacionada `telefonos_usuario`.

### 2. Segunda Forma Normal (2NF): Dependencia Funcional Completa
* **Regla**: Debe cumplir con la 1NF y **todas las columnas no clave deben depender por completo de la clave primaria completa**, y no de una parte de ella (aplicable a claves primarias compuestas).
* **Ejemplo incorrecto**: Clave compuesta `(pedido_id, producto_id)`. Si añadimos la columna `nombre_tienda_proveedor`, esta solo depende de `producto_id`, no del pedido completo.
* **Solución**: Mover la información del proveedor a su propia tabla `productos` o `proveedores`.

### 3. Tercera Forma Normal (3NF): Eliminar Dependencias Transitivas
* **Regla**: Debe cumplir con la 2NF y **ninguna columna no clave debe depender de otra columna no clave** (las columnas no clave deben depender exclusivamente de la clave primaria).
* **Ejemplo incorrecto**: Tabla `usuarios` con columnas `id`, `nombre`, `codigo_postal`, `ciudad`. La columna `ciudad` no depende directamente del `id` del usuario; depende transitoriamente de `codigo_postal`.
* **Solución**: Crear una tabla `codigos_postales` relacionando el código con la ciudad.

### 4. Forma Normal de Boyce-Codd (BCNF): La Regla Definitiva
* **Regla**: Una versión más estricta de la 3NF. Establece que **para toda dependencia funcional no trivial $X \rightarrow Y$, $X$ debe ser una superclave (clave candidata)**. Resuelve anomalías cuando existen claves compuestas que se solapan.
* **Caso clásico**: Si una tabla tiene una clave compuesta por dos atributos y un atributo no clave que determina parcialmente uno de los atributos de la clave, la tabla no está en BCNF.

---

## 7.2 Desnormalización Estratégica en Producción

Aunque la pureza académica aboga por normalizar hasta la 3NF o BCNF de forma dogmática, en el mundo real de alto rendimiento esto puede ser contraproducente. Si para mostrar la pantalla de perfil del usuario necesitas hacer un `JOIN` de 10 tablas separadas (`usuarios`, `direcciones`, `paises`, `telefonos`, `preferencias`, `planes`, `roles`, etc.), el planificador de consultas de PostgreSQL gastará una cantidad masiva de CPU calculando el plan de uniones en cada petición.

La **desnormalización estratégica** consiste en duplicar datos de forma deliberada y controlada en una tabla para acelerar lecturas críticas.
* **Caso común**: Almacenar el campo `total_pedidos` o `saldo_actual` directamente en la tabla `usuarios` en lugar de calcular un `SUM()` sobre millones de registros transaccionales en cada consulta.
* **La regla de oro**: Solo debes desnormalizar si cuentas con un mecanismo automático e infalible (como **Triggers** o transacciones atómicas estrictas) que garantice que los datos duplicados permanezcan 100% sincronizados en caliente.

---

> [!NOTE]
> ### 🧳 El Armario Inteligente de Ropa Clasificada
> 
> Entendamos la normalización, la desnormalización y la elección de JSONB vs. EAV mediante una analogía cotidiana:
> 
> - **La Normalización Estricta (El Armario Organizado por Cajones)**:
>   - Imagina que tienes un armario inteligente en casa. Decides ordenarlo con la máxima disciplina relacional:
>     - Un cajón exclusivo para calcetines individuales.
>     - Otro cajón para camisetas dobladas.
>     - Otro colgador exclusivo para pantalones.
>   - Esto es **normalización limpia**. Nunca tendrás calcetines perdidos entre los pantalones (consistencia total). 
>   - Sin embargo, para vestirte por la mañana (**hacer una consulta de perfil**), tienes que abrir y cerrar 6 cajones separados y unir las prendas en tu cama (**múltiples JOINs**). Si tienes prisa, abrir tantos cajones te hará llegar tarde al trabajo.
> 
> - **La Desnormalización Estratégica (El Kit de Ropa de Emergencia)**:
>   - Para evitar abrir 6 cajones todas las mañanas, decides tomar un pantalón, una camiseta y un par de calcetines y colocarlos juntos en un gancho especial justo en la entrada de tu armario (**Desnormalización**).
>   - Vestirte toma 2 segundos (lectura ultrarrápida en una sola tabla). 
>   - Pero cuidado: si decides cambiar de camiseta favorita, debes recordar actualizar tanto el cajón interno de camisetas como el gancho especial de la entrada. Si lo olvidas, tu kit de ropa estará desincronizado (inconsistencia de datos).
> 
> - **La Catástrofe de EAV (El Fichero de Etiquetas Adhesivas Infinitas)**:
>   - Imagina que compras una prenda nueva muy exótica (por ejemplo, una chaqueta de motociclista con luces LED, baterías y puerto USB). No cabe en tus cajones estándar.
>   - Si usas el patrón **EAV (Entity-Attribute-Value)**, decides pegar una etiqueta adhesiva en tu pared por cada característica:
>     - *Etiqueta 1*: Prenda 45 -> Atributo: "Tipo" -> Valor: "Chaqueta"
>     - *Etiqueta 2*: Prenda 45 -> Atributo: "Color" -> Valor: "Negro"
>     - *Etiqueta 3*: Prenda 45 -> Atributo: "Bateria" -> Valor: "Litio"
>   - Si tienes 100 prendas y cada una tiene 10 propiedades dinámicas, tu pared se llenará de 1,000 notas adhesivas sueltas. Cuando quieras saber qué chaquetas negras con batería tienes en tu armario, tendrás que escanear y fusionar físicamente cientos de etiquetas pequeñas sobre la pared (**unión de filas recursivas pesadas**). Tu pared será un caos ilegible y lento.
> 
> - **El Enfoque Moderno JSONB (La Caja Transparente en el Estante)**:
>   - En lugar de pegar etiquetas sueltas, decides colocar una caja plástica transparente en un estante de tu armario (**Columna JSONB**).
>   - Dentro de esa caja, guardas la chaqueta junto con un manual que describe detalladamente todos sus atributos dinámicos en un formato estructurado y legible. 
>   - Tu armario mantiene su estructura limpia externa (un cajón por prenda), pero tiene la flexibilidad de albergar cualquier cantidad de accesorios dinámicos sin degradar la velocidad y sin llenar la pared de papelitos sueltos.

---

## 7.3 Modelado de Atributos Dinámicos: EAV vs. JSONB

En aplicaciones modernas (como e-commerce de productos variados, sistemas médicos o SaaS personalizables), a menudo necesitamos almacenar atributos dinámicos que varían según el registro.

### 1. El Patrón EAV (Entity-Attribute-Value)
Estructura tradicional que utiliza tres columnas físicas:
* **Entidad (Entity)**: El ID del objeto (ej: `producto_id`).
* **Atributo (Attribute)**: El nombre de la propiedad (ej: `resolucion_pantalla`).
* **Valor (Value)**: El dato en formato de cadena (ej: `"4K"`).

#### ¿Por qué evitarlo hoy en día?
* **Pérdida de tipado estricto**: Todo se guarda como texto (`VARCHAR`), obligando a castear datos en la aplicación.
* **Consultas sumamente complejas**: Para reconstruir un objeto con 5 atributos dinámicos, debes hacer 5 uniones de tabla `JOIN` sobre sí misma.
* **Bajísimo rendimiento**: Destruye la caché del motor y los optimizadores físicos de consultas.

### 2. La Revolución de JSONB en PostgreSQL
PostgreSQL ofrece soporte de primer nivel para datos documentales en formato binario estructurado (**`JSONB`**).
* **Flexibilidad absoluta**: Permite anidar objetos, arreglos y tipos dinámicos.
* **Índices GIN integrados**: Permite indexar internamente las llaves y valores del documento JSONB, logrando búsquedas ultrarrápidas de propiedades internas en microsegundos sin hacer ningún `JOIN`.
* **Validación en aplicación**: Podemos validar y tipar estrictamente el contenido JSONB en el backend (ej: con librerías como Zod o TypeScript) antes de persistirlo.

---

## 7.4 Implementación en TypeScript de Guardado y Validación JSONB

A continuación, implementaremos un servicio en TypeScript que modela y almacena registros de un e-commerce con catálogo dinámico utilizando una columna **JSONB** en PostgreSQL. Validaremos los atributos dinámicos con la potente librería **Zod** antes de la inserción y demostraremos consultas nativas con operadores de búsqueda sobre JSONB:

#### [gestionCatalogo.ts](file:///Users/andres/Documents/biblioteca/sql-databases-programming-book/src/services/gestionCatalogo.ts)
```typescript
import { z } from 'zod';
import { dbPool } from '../clients/dbClient';

// Schema estricto de validación usando Zod para las especificaciones dinámicas de los productos
export const EspecificacionesProductoSchema = z.object({
  marca: z.string().min(1),
  modelo: z.string().min(1),
  pesoGramos: z.number().positive().optional(),
  dimensiones: z.object({
    altoCm: z.number(),
    anchoCm: z.number(),
    profundidadCm: z.number()
  }).optional(),
  atributosExtra: z.record(z.union([z.string(), z.number(), z.boolean()])).default({})
});

export type EspecificacionesProducto = z.infer<typeof EspecificacionesProductoSchema>;

export interface Producto {
  id?: number;
  nombre: string;
  precio: number;
  categoria: string;
  especificaciones: EspecificacionesProducto;
}

// 1. Guardar producto validando estrictamente su payload JSONB
export async function registrarProducto(producto: Producto): Promise<number> {
  // Validamos los datos dinámicos en tiempo de ejecución antes de tocar la base de datos
  const especificacionesValidadas = EspecificacionesProductoSchema.parse(producto.especificaciones);

  const cliente = await dbPool.connect();

  try {
    const sqlInsertar = `
      INSERT INTO productos (nombre, precio, categoria, especificaciones)
      VALUES ($1, $2, $3, $4::jsonb)
      RETURNING id
    `;

    const resultado = await cliente.query(sqlInsertar, [
      producto.nombre,
      producto.precio,
      producto.categoria,
      JSON.stringify(especificacionesValidadas)
    ]);

    return resultado.rows[0].id;

  } catch (error: any) {
    console.error('Error al registrar el producto dinámico:', error.message);
    throw error;
  } finally {
    cliente.release();
  }
}

// 2. Buscar productos mediante operadores de filtrado JSONB nativos
export async function buscarProductosPorEspecificaciones(
  categoria: string,
  marca: string,
  minAltoCm: number
): Promise<Producto[]> {
  const cliente = await dbPool.connect();

  try {
    // Operadores JSONB clave en PostgreSQL:
    // - `->>` obtiene un valor de texto de una propiedad interna
    // - `->` obtiene el objeto JSON interno
    // - `@>` comprueba si un JSON contiene a otro fragmento JSON (indexable por GIN)
    const sqlBuscar = `
      SELECT id, nombre, precio, categoria, especificaciones
      FROM productos
      WHERE categoria = $1
        AND especificaciones @> $2::jsonb
        AND CAST(especificaciones->'dimensiones'->>'altoCm' AS NUMERIC) >= $3
    `;

    // Filtramos que contenga un fragmento de coincidencia exacta de marca
    const filtroContiene = JSON.stringify({ marca });

    const resultado = await cliente.query(sqlBuscar, [categoria, filtroContiene, minAltoCm]);

    return resultado.rows.map(row => ({
      id: row.id,
      nombre: row.nombre,
      precio: parseFloat(row.precio),
      categoria: row.categoria,
      especificaciones: row.especificaciones // PostgreSQL lo parsea automáticamente a objeto de JS
    }));

  } catch (error: any) {
    console.error('Error al consultar productos por JSONB:', error.message);
    throw error;
  } finally {
    cliente.release();
  }
}
```

---

## Resumen del Capítulo

* Las **Formas Normales** (1NF a BCNF) definen directrices lógicas rigurosas para erradicar la redundancia y prevenir anomalías de actualización de datos.
* La **Desnormalización Estratégica** sacrifica deliberadamente la pureza estructural duplicando datos puntuales para acelerar lecturas críticas, requiriendo sincronización atómica estricta (ej. triggers o transacciones).
* El patrón de diseño clásico **EAV (Entity-Attribute-Value)** degrada el rendimiento de la CPU debido a múltiples uniones costosas y carece de tipado nativo.
* El soporte **JSONB** nativo en motores modernos como PostgreSQL ofrece lo mejor de dos mundos: la flexibilidad del desarrollo documental y la velocidad de indexación de un motor relacional consolidado.

En el próximo capítulo, aprenderemos cómo diagnosticar cuellos de botella microscópicos de forma científica mediante la interpretación de planes de consulta usando **EXPLAIN ANALYZE** y el estudio interno de algoritmos físicos de uniones (**Joins**).

---

[← Capítulo anterior (Capítulo 6)](06-ctes-y-window-functions.md) | [Inicio (README.md)](README.md) | [Capítulo siguiente (Capítulo 8) →](08-explain-analyze-y-joins.md)
