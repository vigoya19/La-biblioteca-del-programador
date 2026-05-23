# Apéndice A: Ejercicios Prácticos Resueltos Paso a Paso

> "La teoría te proporciona el mapa del territorio; pero solo la resolución de problemas reales te enseña a conducir en la tormenta."

Este apéndice está diseñado para poner a prueba tu capacidad de razonamiento arquitectónico y tus habilidades de programación en el ecosistema NoSQL. Encontrarás cuatro desafíos reales de ingeniería que van **desde el nivel de entrada (Fácil) hasta problemas de diseño avanzados (Experto)**. 

Cada ejercicio incluye la descripción del problema, el análisis paso a paso de la estrategia física, y la implementación de código completa en **TypeScript con tipado estricto**.

---

## 🧭 Desafío 1 (Fácil): Limitador de Tasa (Rate Limiter) en Redis

### 1. El Planteamiento del Problema
Tu startup de SaaS está lanzando una API pública. Para evitar abusos, ataques DDoS de denegación de servicio o raspado (*scraping*) malicioso de tu contenido, debes proteger tu API limitando el tráfico: **cada dirección IP de cliente puede realizar un máximo de 100 peticiones por minuto**. Si un cliente supera este umbral, el servidor Express debe responder con el código de estado HTTP `429 Too Many Requests`.

### 2. Análisis Arquitectónico Paso a Paso
Para resolver este problema con latencias de microsegundos, la base de datos RAM **Redis** es la herramienta idónea mediante el patrón de **Ventana Fija (Fixed Window)**:

* **Paso 1 (Diseño de Claves)**: Generamos una clave única en memoria RAM utilizando la IP del cliente y la ventana de tiempo actual en minutos (para aislar las ventanas limpias de tiempo):
  $$\text{Clave} = \text{"ratelimit:192.168.1.50:"} + \text{MinutoActual}$$
* **Paso 2 (Atomicidad)**: Cuando llega una petición, no podemos ejecutar dos comandos separados (leer y luego escribir) porque provocaría carreras de concurrencia. Utilizamos una **Transacción Atómica (MULTI/EXEC)** en Redis para:
  1. Incrementar el contador de la clave mediante el comando `INCR`.
  2. Asignar un tiempo de vida (TTL) de 60 segundos mediante el comando `EXPIRE` para que Redis limpie automáticamente la memoria al pasar el minuto.
* **Paso 3 (Decisión)**: Si el resultado devuelto por `INCR` es mayor a 100, bloqueamos la conexión del usuario.

### 3. Solución de Código en TypeScript:

#### [rateLimiter.ts](file:///Users/andres/Documents/biblioteca/nosql-databases-programming-book/src/ejercicios/rateLimiter.ts)
```typescript
import Redis from 'ioredis';
import { Request, Response, NextFunction } from 'express';

const redis = new Redis({ host: '127.0.0.1', port: 6379 });

export async function middlewareRateLimiter(req: Request, res: Response, next: NextFunction): Promise<void> {
  const ipCliente = req.ip || 'ip_desconocida';
  
  // Obtener el minuto actual del sistema para segmentar la ventana temporal
  const minutoActual = new Date().getMinutes();
  const claveRedis = `ratelimit:${ipCliente}:${minutoActual}`;

  try {
    // 1. Crear una transacción MULTI para ejecutar los comandos atómicamente
    const pipeline = redis.multi();
    pipeline.incr(claveRedis);
    pipeline.expire(claveRedis, 60); // Caducar en 1 minuto

    // 2. Ejecutar la transacción en Redis
    const respuestas = await pipeline.exec();
    
    if (!respuestas || respuestas.length === 0) {
      res.status(500).json({ error: 'Fallo al inicializar transacciones en caché.' });
      return;
    }

    // El resultado del INCR reside en la primera posición del array de retorno
    // Respuestas en ioredis devuelven [error, resultado] por cada comando
    const totalPeticiones = respuestas[0][1] as number;

    // 3. Evaluar el umbral limitador
    if (totalPeticiones > 100) {
      res.status(429).json({
        error: 'Too Many Requests',
        message: 'Has superado el límite de 100 peticiones por minuto. Inténtalo más tarde.'
      });
      return;
    }

    next();
  } catch (error) {
    console.error('[RateLimiter-Error] Fallo en el middleware:', error);
    // En caso de fallo crítico en Redis, dejamos pasar la petición para no bloquear el negocio (fail-open)
    next();
  }
}
```

---

## 🧭 Desafío 2 (Medio): Pipeline de Métricas Cruzadas en MongoDB

### 1. El Planteamiento del Problema
Administras el catálogo analítico de un e-commerce. Tienes dos colecciones en MongoDB: `productos` (que almacena datos como nombre, precio y categoría) y `ventas` (que almacena los IDs de productos comprados y la cantidad vendida). 
Necesitas construir una consulta analítica de alto rendimiento que:
1. Agrupe los productos por **Categoría**.
2. Sume los ingresos totales generados por cada categoría (precio del producto multiplicado por cantidad vendida).
3. Filtre y devuelva únicamente las categorías cuyos ingresos totales **superen los $10,000 USD**.
4. Ordene el listado de mayor a menor ingreso.

### 2. Análisis Arquitectónico Paso a Paso
Para procesar millones de registros de forma óptima directamente en la base de datos sin sobrecargar la RAM de tu backend, empleamos el **Aggregation Framework**:

* **Paso 1 (`$lookup`)**: Cruzamos la colección de `ventas` con `productos` mediante un JOIN documental para traer la información del precio y categoría del producto.
* **Paso 2 (`$unwind`)**: Desglosamos el array de productos cruzado para convertirlo en un objeto plano manejable.
* **Paso 3 (`$group`)**: Agrupamos por la clave `$categoria`. Calculamos los ingresos totales multiplicando la cantidad de la venta por el precio del producto mediante la operación aritmética `$multiply` sumada con `$sum`.
* **Paso 4 (`$match`)**: Filtramos el resultado agrupado reteniendo solo valores superiores a 10,000.
* **Paso 5 (`$sort`)**: Ordenamos de forma descendente (`-1`).

### 3. Solución de Código en TypeScript:

#### [analisisVentas.ts](file:///Users/andres/Documents/biblioteca/nosql-databases-programming-book/src/ejercicios/analisisVentas.ts)
```typescript
import { Schema, model, Document } from 'mongoose';

// Interfaces físicas
export interface IProducto extends Document {
  nombre: string;
  precio: number;
  categoria: string;
}

export interface IVenta extends Document {
  productoId: Schema.Types.ObjectId;
  cantidad: number;
  fecha: Date;
}

const ProductoSchema = new Schema<IProducto>({
  nombre: { type: String, required: true },
  precio: { type: Number, required: true },
  categoria: { type: String, required: true }
});

const VentaSchema = new Schema<IVenta>({
  productoId: { type: Schema.Types.ObjectId, ref: 'Producto', required: true },
  cantidad: { type: Number, required: true },
  fecha: { type: Date, default: Date.now }
});

export const Producto = model<IProducto>('Producto', ProductoSchema);
export const Venta = model<IVenta>('Venta', VentaSchema);

interface CategoriasExitosas {
  categoria: string;
  ingresosTotales: number;
}

// 4. Implementación de la agregación analítica de alta velocidad
export async function obtenerCategoriasMasVendidas(): Promise<CategoriasExitosas[]> {
  const pipeline = [
    // Etapa 1: Cruzar Ventas con Productos
    {
      $lookup: {
        from: 'productos',          // Nombre de la colección física en MongoDB
        localField: 'productoId',   // Campo de la colección Ventas
        foreignField: '_id',        // Campo de la colección Productos
        as: 'detalleProducto'
      }
    },
    // Etapa 2: Aplanar el array de detalleProducto generado por $lookup
    { $unwind: '$detalleProducto' },
    
    // Etapa 3: Agrupar por categoría y calcular ingresos de forma aritmética
    {
      $group: {
        _id: '$detalleProducto.categoria',
        ingresosTotales: {
          $sum: { $multiply: ['$cantidad', '$detalleProducto.precio'] }
        }
      }
    },
    
    // Etapa 4: Filtrar categorías que superen el umbral de 10,000 USD
    {
      $match: {
        ingresosTotales: { $gt: 10000 }
      }
    },
    
    // Etapa 5: Ordenar descendente por ingresos
    { $sort: { ingresosTotales: -1 } }
  ];

  const resultados = await Venta.aggregate(pipeline);
  
  return resultados.map(res => ({
    categoria: res._id,
    ingresosTotales: res.ingresosTotales
  }));
}
```

---

## 🧭 Desafío 3 (Avanzado): Single-Table Design para un Foro en DynamoDB

### 1. El Planteamiento del Problema
Debes diseñar el motor de almacenamiento de una plataforma de foros (como Reddit o StackOverflow) utilizando **Amazon DynamoDB Single-Table Design**. El sistema debe representar a la perfección tres entidades: **Canales**, **Hilos** de discusión de los canales, y **Comentarios** anidados dentro de cada hilo. 

Debes poder resolver las siguientes consultas de forma instantánea ($O(1)$) en una sola llamada de red sin JOINs:
1. Buscar el perfil y metadatos de un Canal.
2. Listar todos los Hilos de discusión de un canal ordenados cronológicamente desde el más nuevo.
3. Listar todos los Comentarios de un hilo de discusión.

### 2. Análisis Arquitectónico Paso a Paso
Para mezclar estas entidades de forma eficiente en una única tabla física, diseñamos un esquema de claves compuesto genérico sobrecargado con prefijos de negocio:

* **Canal**:
  * `PK`: `CANAL#<canalId>`
  * `SK`: `METADATA`
* **Hilo**:
  * `PK`: `CANAL#<canalId>`  *(Permite agrupar los hilos bajo la misma partición del canal)*
  * `SK`: `HILO#<timestamp>#<hiloId>` *(El timestamp intermedio permite que DynamoDB ordene físicamente los hilos de forma cronológica natural en disco)*
* **Comentario**:
  * `PK`: `HILO#<hiloId>` *(Los comentarios de un hilo se agrupan en su propia partición aislada)*
  * `SK`: `COMENTARIO#<timestamp>#<comentarioId>`

```
    Estructura Visual de Datos en la misma Tabla Física
    ┌──────────────────────┬──────────────────────────────────┬────────────────────────┐
    │ PK (Partition Key)   │ SK (Sort Key)                    │ Atributos de Negocio   │
    ├──────────────────────┼──────────────────────────────────┼────────────────────────┤
    │ CANAL#tecnologia     │ METADATA                         │ nombre: "Tecnología"   │
    │ CANAL#tecnologia     │ HILO#2026-05-22T21:00Z#hilo_901  │ titulo: "Lanzamiento Go"│
    │ CANAL#tecnologia     │ HILO#2026-05-23T08:00Z#hilo_902  │ titulo: "React 19"     │
    │ HILO#hilo_902        │ COMENTARIO#2026-05-23T08:05Z#c_1 │ texto: "Genial!"       │
    └──────────────────────┴──────────────────────────────────┴────────────────────────┘
```

* **Cómo responder las consultas**:
  * *Consulta 1 (Canal)*: Query donde `PK = "CANAL#tecnologia"` y `SK = "METADATA"`.
  * *Consulta 2 (Hilos del canal)*: Query donde `PK = "CANAL#tecnologia"`, con la condición `SK begins_with("HILO#")` y orden inverso (`ScanIndexForward = false`) para ordenar cronológicamente de forma descendente.
  * *Consulta 3 (Comentarios)*: Query donde `PK = "HILO#hilo_902"` y `SK begins_with("COMENTARIO#")`.

### 3. Solución de Código en TypeScript:

#### [foroDynamoService.ts](file:///Users/andres/Documents/biblioteca/nosql-databases-programming-book/src/ejercicios/foroDynamoService.ts)
```typescript
import { DynamoDBClient } from '@aws-sdk/client-dynamodb';
import { DynamoDBDocumentClient, QueryCommand } from '@aws-sdk/lib-dynamodb';

const clienteBase = new DynamoDBClient({ region: 'us-east-1' });
const dynamoDb = DynamoDBDocumentClient.from(clienteBase);

const NOMBRE_TABLA = 'ForoTablaUnica';

interface HiloForo {
  hiloId: string;
  titulo: string;
  autor: string;
  fechaCreacion: string;
}

// 4. Buscar hilos ordenados de forma descendente cronológica
export async function listarHilosDeCanal(canalId: string): Promise<HiloForo[]> {
  const comando = new QueryCommand({
    TableName: NOMBRE_TABLA,
    KeyConditionExpression: 'PK = :pk AND SK BEGINS_WITH(:skPrefijo)',
    ExpressionAttributeValues: {
      ':pk': `CANAL#${canalId}`,
      ':skPrefijo': 'HILO#'
    },
    // ScanIndexForward = false indica orden descendente (de nuevo a viejo)
    ScanIndexForward: false
  });

  const respuesta = await dynamoDb.send(comando);

  if (!respuesta.Items) return [];

  return respuesta.Items.map(item => {
    // El SK tiene el formato "HILO#<timestamp>#<hiloId>"
    const partesSk = item.SK.split('#');
    const hiloId = partesSk[2];

    return {
      hiloId,
      titulo: item.titulo,
      autor: item.autor,
      fechaCreacion: item.fechaCreacion
    };
  });
}
```

---

## 🧭 Desafío 4 (Experto): Detección de Fraude Circular en Neo4j

### 1. El Planteamiento del Problema
Eres el director de ciberseguridad y prevención de lavado de dinero de un banco multinacional. Los delincuentes financieros suelen mover fondos a través de intrincadas redes circulares para camuflar el origen del dinero: la Cuenta A transfiere a la Cuenta B, la B transfiere a la C, y la C transfiere de regreso a la Cuenta A en un bucle cerrado. 

Debes diseñar una consulta analítica en **Cypher** que detecte automáticamente estos **anillos de fraude circulares de entre 3 y 4 niveles de longitud**, e implementar un servicio en TypeScript que al detectar un bucle congele automáticamente la transacción mediante lógica distribuida.

### 2. Análisis Arquitectónico Paso a Paso
Este problema es extremadamente complejo de procesar en bases relacionales SQL tradicionales porque requiere buscar rutas recursivas de caminos infinitos cruzando millones de registros de transacciones. 

* **Paso 1 (La Query Cypher)**: Aprovechando la adyacencia libre de índices de Neo4j, definimos un patrón de camino circular que empiece y termine en la misma cuenta de origen:
  ```cypher
  MATCH ruta = (c1:Cuenta)-[:TRANSFERIDO*3..4]->(c1)
  RETURN c1.numero AS cuentaSospechosa, nodes(ruta) AS cuentasAnillo, relationships(ruta) AS transferencias
  ```
  El operador `*3..4` le ordena a Neo4j buscar recursivamente en el grafo caminos de longitud de entre 3 y 4 saltos físicos conectando de vuelta con la cuenta de origen `c1`.
* **Paso 2 (Compensación)**: Si detectamos que la transferencia que se está intentando registrar completa un bucle sospechoso de fraude, congelamos las cuentas y arrojamos un error lógico.

### 3. Solución de Código en TypeScript:

#### [detectorFraude.ts](file:///Users/andres/Documents/biblioteca/nosql-databases-programming-book/src/ejercicios/detectorFraude.ts)
```typescript
import neo4j from 'neo4j-driver';

const driver = neo4j.driver('bolt://localhost:7687', neo4j.auth.basic('neo4j', 'seguridad'));

interface ResultadoFraude {
  cuentaOrquestadora: string;
  anilloCuentas: string[];
}

// 4. Analizar si registrar una nueva transferencia completa una red circular de fraude
export async function validarYRegistrarTransferencia(
  origen: string, 
  destino: string, 
  monto: number
): Promise<boolean> {
  const session = driver.session();

  // Transacción segura de escritura
  const tx = await session.beginTransaction();

  try {
    // A. Crear la relación de la transferencia en caliente
    const queryCrear = `
      MERGE (a:Cuenta {numero: $origen})
      MERGE (b:Cuenta {numero: $destino})
      CREATE (a)-[t:TRANSFERIDO {monto: $monto, fecha: datetime()}]->(b)
    `;
    await tx.run(queryCrear, { origen, destino, monto });

    // B. Buscar si esta nueva transferencia acaba de cerrar un bucle circular de fraude
    const queryDeteccion = `
      MATCH ruta = (c1:Cuenta {numero: $origen})-[:TRANSFERIDO*3..4]->(c1)
      RETURN c1.numero AS cuentaOrquestadora, [n IN nodes(ruta) | n.numero] AS nombresCuentas
      LIMIT 1
    `;
    
    const resultado = await tx.run(queryDeteccion, { origen });

    if (resultado.records.length > 0) {
      // ¡RED CIRCULAR DE FRAUDE DETECTADA!
      const record = resultado.records[0];
      const cuentasAfectadas = record.get('nombresCuentas');

      console.warn(`[ALERTA-FRAUDE] Bucle de lavado de dinero detectado: ${cuentasAfectadas.join(' -> ')}`);
      
      // Aplicamos compensación lógica: Hacemos un ROLLBACK físico de la transacción en Neo4j
      await tx.rollback();
      return false; // Transferencia bloqueada por fraude
    }

    // Si todo está seguro, consolidamos la transacción física en el grafo
    await tx.commit();
    return true; // Transferencia autorizada y guardada
  } catch (error) {
    await tx.rollback();
    throw error;
  } finally {
    await session.close();
  }
}
```

---

[← Volver al proyecto integrador (Capítulo 12)](12-proyecto-practico.md) | [Volver al Inicio (README.md)](README.md)
