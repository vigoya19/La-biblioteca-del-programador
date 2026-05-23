# Capítulo 9: Indexación Avanzada y Búsqueda Full-Text

> "Una consulta rápida no es aquella que recorre millones de registros a la velocidad de la luz; es aquella que, gracias a un diseño inteligente de índices, sabe omitir físicamente el 99.99% de la base de datos."

El rendimiento óptimo de cualquier base de datos NoSQL reside en su capacidad para reducir el trabajo de Entrada/Salida (I/O) en disco. Cuando realizas una consulta y el motor de datos se ve obligado a leer secuencialmente cada uno de los archivos físicos de tu disco duro buscando coincidencias, ocurre la peor pesadilla de un DBA: un **COLLSCAN (Collection Scan - Escaneo de Colección Completo)**. 

Para erradicar esto, las bases de datos NoSQL proveen un catálogo versátil de índices. En este capítulo, desmitificaremos el diseño avanzado de índices compuestos aplicando la regla dorada de **Igualdad-Orden-Rango (ESR)**, estudiaremos los índices geoespaciales y TTL, y analizaremos cómo integrar nuestras bases de datos con motores de búsqueda dedicados como **Elasticsearch** y **OpenSearch** mediante arquitecturas CDC (Change Data Capture).

---

## 9.1 Índices Compuestos en MongoDB y la Regla ESR

Un **Índice Compuesto** es una estructura ordenada en forma de árbol B-Tree que consolida múltiples campos de un documento en una sola llave de búsqueda en disco. 

Para que un índice compuesto funcione al máximo rendimiento y permita resolver tanto filtros lógicos como ordenamientos en memoria de forma transparente, los campos del índice deben seguir obligatoriamente la **Regla de Orden ESR (Equality, Sort, Range)**:

```
                  ┌─────────────────────────────────────┐
                  │ 1. EQUALITY (Igualdad Exacta)       │  ──► Filtros directos (ej: activo = true)
                  └──────────────────┬──────────────────┘
                                     │
                                     ▼
                  ┌─────────────────────────────────────┐
                  │ 2. SORT (Ordenamiento del Índice)   │  ──► Orden de salida (ej: precio desc)
                  └──────────────────┬──────────────────┘
                                     │
                                     ▼
                  ┌─────────────────────────────────────┐
                  │ 3. RANGE (Filtros de Rangos)        │  ──► Comparaciones (ej: edad > 18)
                  └─────────────────────────────────────┘
```

1. **Equality (Igualdad)**: Coloca en primer lugar los campos que buscas mediante comparaciones de igualdad exacta (por ejemplo, `{ activo: true, rol: "admin" }`). Esto reduce el espacio de búsqueda en un B-Tree al instante.
2. **Sort (Ordenamiento)**: Coloca en segundo lugar los campos por los cuales ordenarás tu resultado. Al estar ordenados dentro del índice físico, el motor de la base de datos devuelve los datos ya listos por orden de disco, evitando el costoso y peligroso **In-Memory Sort (Ordenamiento en memoria)** que causa fallos si el buffer supera los 32 MB.
3. **Range (Rango)**: Coloca al final los campos sobre los cuales aplicarás búsquedas comparativas de rango (por ejemplo, `{ edad: { $gt: 18 } }`, fechas `$gte` o `$lte`). 
   * *¿Por qué al final?* Si colocas un campo de rango en medio del índice, el B-Tree se bifurcará en múltiples ramas para los valores menores/mayores, impidiendo que el motor pueda usar las siguientes claves del índice compuesto para el ordenamiento o las igualdades consecutivas.

---

## 9.2 Índices Especializados: Geoespaciales y TTL

### 1. Índices Geoespaciales (`2dsphere`)
MongoDB admite búsquedas espaciales basadas en las coordenadas físicas de la Tierra utilizando el formato estándar **GeoJSON**. Los índices de tipo `2dsphere` permiten calcular distancias geodésicas en una esfera tridimensional:
* **Consultas `$near` (Cercanía)**: Encuentra restaurantes en un radio de 500 metros alrededor de la ubicación actual del usuario mediante coordenadas de Longitud y Latitud.
* **Consultas `$geoWithin` (Contención)**: Encuentra propiedades o repartidores ubicados dentro del polígono delimitador de un barrio o una ciudad específica.

### 2. Índices TTL (Time-To-Live)
Estructuras de datos que expiran y **eliminan de forma automática documentos del disco duro** transcurrido un intervalo de tiempo específico.
* **¿Cómo funcionan?** MongoDB tiene un hilo de fondo (*background thread*) que se ejecuta de forma asíncrona cada 60 segundos. Este hilo busca documentos cuyas claves TTL hayan expirado y los borra físicamente sin requerir que tu servidor backend ejecute scripts o CRON jobs de limpieza. Es ideal para sesiones de usuario volátiles, carritos de compra temporales, logs de auditoría o tokens de seguridad.

> [!NOTE]
> ### 🔍 El Índice Alfabético Temático del Libro y los Guardias con Brújula
> 
> Visualicemos el dilema de la indexación:
> 
> - **El COLLSCAN (Sin Índices)**: Imagina que compras una enciclopedia física de 5,000 páginas. Te pido que cuentes cuántas veces se menciona la palabra "Jirafas". Al no tener índice, debes sentarte y leer de forma secuencial cada una de las 5,000 páginas, línea por línea, con una lupa en la mano. Te tomará 3 semanas resolver una simple consulta de lectura.
> - **El Índice Compuesto con Regla ESR**:
>   - Diseñamos un **Índice Temático al final del Libro**.
>   - **Equality (Igualdad)**: Buscas bajo la letra "J" de Jirafas. Saltas al instante a la sección de Jirafas omitiendo el 99% del libro.
>   - **Sort (Ordenamiento)**: Dentro de "Jirafas", los subtemas están ordenados alfabéticamente por región: *África, Asia, Europa*. Vas directo a *África* sin rebuscar.
>   - **Range (Rango)**: Finalmente, ves un rango de fechas históricas de avistamientos: *1990 a 2026*. Buscas en ese rango delimitado. Encontraste la página exacta en 2 segundos.
> - **Índice Geoespacial**: Imagina un **Tablero de Dardos con una Cuadrícula y una Brújula**. El índice `2dsphere` divide al mundo en pequeñas cuadrículas numeradas en base a coordenadas. Cuando le pides al sistema buscar lo que hay cerca de ti, el sistema toma un compás físico, traza un círculo alrededor de tu coordenada de cuadrícula y solo lee las carpetas de las cuadrículas afectadas por el círculo, ignorando por completo el resto de las carpetas de los otros continentes del planeta.

---

## 9.3 Integración con Motores de Búsqueda Dedicados: Elasticsearch

A pesar de la riqueza de índices de MongoDB o DynamoDB, las bases de datos transaccionales NoSQL no fueron diseñadas para búsquedas de texto predictivas de nivel comercial (búsqueda difusa de palabras, autocompletado inteligente con errores ortográficos, o ranking de relevancia de documentos mediante algoritmos **TF-IDF** o **BM25**).

Para lograr esto, las arquitecturas modernas acoplan la base de datos NoSQL a un motor de búsqueda full-text dedicado como **Elasticsearch** u **OpenSearch** empleando el patrón **CDC (Change Data Capture)**:

```
 ┌───────────────┐                  ┌────────────────┐                  ┌───────────────┐
 │   MongoDB     │ ───────────────► │ Change Stream  │ ───────────────► │ Elasticsearch │
 │  (Escritura)  │  (CDC Worker)    │  (Event Log)   │   (Sincronía)    │  (Busquedas)  │
 └───────────────┘                  └────────────────┘                  └───────────────┘
```

* **Change Streams**: MongoDB permite escuchar en tiempo real sus logs de operaciones internas (`oplog`) de forma segura. Cada vez que un documento se inserta, modifica o elimina de MongoDB, el motor emite un evento en caliente. Un Worker de integración captura el evento y sincroniza el documento de forma asíncrona hacia Elasticsearch en pocos milisegundos, manteniendo ambos mundos en perfecta armonía.

---

## 9.4 Implementación en TypeScript de Sincronía CDC a Elasticsearch

Implementemos un listener en caliente utilizando **Change Streams** de MongoDB para sincronizar automáticamente cambios hacia un cliente simulado de Elasticsearch en TypeScript:

#### [sincronizadorElastic.ts](file:///Users/andres/Documents/biblioteca/nosql-databases-programming-book/src/services/sincronizadorElastic.ts)
```typescript
import { MongoClient, ChangeStream } from 'mongodb';

// 1. Clientes simulados de conexión
const mongoClient = new MongoClient('mongodb://localhost:27017');

class ClientElasticsearch {
  async indexarDocumento(id: string, documento: any): Promise<void> {
    console.log(`[Elasticsearch] Documento indexado: ID = ${id}`, documento);
  }

  async eliminarDocumento(id: string): Promise<void> {
    console.log(`[Elasticsearch] Documento eliminado: ID = ${id}`);
  }
}

const elasticsearch = new ClientElasticsearch();

// 2. Iniciar el monitoreo continuo de cambios (Change Stream)
export async function iniciarSincronizacionCDC(): Promise<void> {
  await mongoClient.connect();
  const db = mongoClient.db('biblioteca_nosql');
  const coleccionProductos = db.collection('productos');

  console.log('[CDC-Worker] Escuchando eventos Change Stream en MongoDB...');

  // Habilitamos un stream de cambios para capturar inserts, updates y deletes
  const changeStream: ChangeStream = coleccionProductos.watch([
    {
      $match: {
        operationType: { $in: ['insert', 'update', 'replace', 'delete'] }
      }
    }
  ], { fullDocument: 'updateLookup' }); // Exige retornar el documento completo tras un update

  // Escuchar eventos en caliente
  changeStream.on('change', async (event: any) => {
    const documentoId = event.documentKey._id.toString();

    try {
      if (event.operationType === 'delete') {
        // Si el documento se borró de MongoDB, lo sacamos de Elasticsearch
        await elasticsearch.eliminarDocumento(documentoId);
      } else {
        // Para insert, update o replace, indexamos el documento completo de MongoDB
        const documentoCompleto = event.fullDocument;
        
        // Sanitizar el objeto eliminando datos internos de MongoDB
        delete documentoCompleto._id;
        
        await elasticsearch.indexarDocumento(documentoId, documentoCompleto);
      }
    } catch (error) {
      console.error(`[CDC-Worker-Error] Fallo al sincronizar ID = ${documentoId}:`, error);
      // Aquí se implementaría una cola de reintentos (Dead Letter Queue)
    }
  });
}
```

---

    } catch (error) {
      console.error(`[CDC-Worker-Error] Fallo al sincronizar ID = ${documentoId}:`, error);
      // Aquí se implementaría una cola de reintentos (Dead Letter Queue)
    }
  });
}
```

---

## 9.5 Deep Dive: B-Tree Splits en WiredTiger y Curvas Hilbert en S2 Geometry

Los índices compuestos y geoespaciales agilizan las lecturas, pero imponen un costo físico inmenso al hardware durante las inserciones masivas de datos:

> [!NOTE]
> ### 📚 La Analogía del Estante de Libros sin Espacio y el Hilo Continuo del Laberinto
> 
> Visualicemos el comportamiento microscópico de los índices B-Tree y la geolocalización esférica en disco:
> 
> - **El "Node Split" (División de Nodo en Índices)** es equivalente a un **Estante Físico de Libros Ajustado**:
>   - Tienes una estantería de madera diseñada para almacenar exactamente 10 libros ordenados de la A a la Z. El estante está lleno.
>   - De pronto, llega un nuevo libro bajo la letra "G". Al no caber físicamente entre la "F" y la "H", no puedes empujar las paredes de madera del estante.
>   - **La Operación**: Debes comprar un **Nuevo Estante Completo**, retirar los 5 libros de la "H" a la "Z" del primer estante, colocarlos en el nuevo y colgar un letrero en el primer estante que diga: *"Para libros de la H a la Z, ver el estante de al lado"*. 
>   - Esta reorganización física es un **Node Split**. Durante los microsegundos que tardas en mover los libros, nadie puede consultar la estantería (**Bloqueo de Escritura**), ralentizando las escrituras concurrentes de tu aplicación.
> - **La Curva Hilbert (Geolocalización `2dsphere`)** es equivalente a **Medir el Planeta con un Único Hilo de Lana**:
>   - Las coordenadas de GPS tienen dos dimensiones independientes: Latitud (Y) y Longitud (X). Pero los índices de bases de datos son unidimensionales (una sola línea recta ordenada de números de menor a mayor). ¿Cómo metes un mapa plano 2D en una sola línea 1D?
>   - **La Solución**: Imagina que tomas un **Hilo de Lana Infinito** y lo pegas sobre un globo terráqueo dibujando un **Laberinto de curvas sumamente intrincado (la Curva de Hilbert)** que pasa y cubre cada metro cuadrado del planeta de forma continua.
>   - Este laberinto tiene una propiedad mágica: dos casas que estén físicamente muy cerca en la calle estarán también muy cerca a lo largo de la distancia de la hebra de hilo de lana extendido.
>   - El motor traduce tu coordenada GPS al milímetro de hilo exacto (el **S2 Cell ID** de 64 bits) y realiza una búsqueda de rango simple en una línea recta ordenada en su índice B-Tree, localizando vecinos en microsegundos.

### 1. Internals de WiredTiger: B-Tree Node Splits
En MongoDB, el motor de almacenamiento **WiredTiger** organiza físicamente los índices en disco utilizando estructuras jerárquicas **B-Tree (Árboles B)** balanceadas.
* Cada nodo físico del árbol (una página de disco de típicamente 16 KB) almacena un rango ordenado de claves e IDs de documentos.
* Cuando realizas inserciones concurrentes masivas en una colección con múltiples índices, WiredTiger debe insertar las nuevas llaves en la página correspondiente en disco de forma ordenada.
* **El Split**: Si la página física está saturada (100% de ocupación de bytes), el motor ejecuta un **Node Split**:
  1. Solicita al sistema operativo un nuevo bloque de almacenamiento en disco de 16 KB.
  2. Divide a la mitad las claves de la página llena original.
  3. Mueve el 50% de las claves al nuevo bloque y actualiza el puntero en la página padre de arriba.
  4. Para garantizar la seguridad ante fallos, escribe esta alteración en el diario de logs (*Journal*) y retiene bloqueos (*Write Locks*) locales que suspenden temporalmente otras lecturas/escrituras en esa rama del índice, causando cuellos de botella por contención si hay demasiados índices activos en la colección.

### 2. Google S2 Geometry Library (Índices `2dsphere`)
Para indexar coordenadas de la Tierra en MongoDB, se utiliza la librería matemática **Google S2**:
* Proyecta la esfera tridimensional del globo terrestre sobre las seis caras planas de un cubo.
* Cada una de las seis caras del cubo se divide de forma jerárquica y recursiva en cuadrículas microscópicas llamadas **S2 Cells** (celdas S2) de hasta 30 niveles de profundidad (donde una celda de nivel 30 mide apenas 1 centímetro cuadrado).
* Para ordenar y recorrer estas celdas bidimensionales en un índice lineal de base de datos unidimensional, la librería traza una **Curva de Hilbert de Relleno Espacial (Space-Filling Curve)** continuo. 
* El resultado de la curva asigna un identificador numérico entero único de 64 bits (**S2 Cell ID**) a cada coordenada. MongoDB guarda este entero unidimensional en su B-Tree regular, permitiendo resolver consultas espaciales complejas de cercanía en un solo salto analítico directo de base de datos.

---

## Resumen del Capítulo


* Para erradicar los lentos **COLLSCAN**, las bases de datos NoSQL emplean árboles ordenados B-Tree configurados mediante índices compuestos.
* Los índices compuestos deben respetar estrictamente la **Regla ESR (Equality, Sort, Range)** para optimizar búsquedas tridimensionales en disco duro.
* MongoDB habilita indexaciones avanzadas para proximidad física esférica mediante **GeoJSON (`2dsphere`)** y depuración automática asíncrona mediante **TTL**.
* Búsquedas predictivas complejas de texto comercial se logran sincronizando de forma asíncrona el motor NoSQL transaccional con **Elasticsearch** mediante el patrón reactivo **CDC (Change Data Capture)**.

En el próximo capítulo, ingresaremos al área de la seguridad informática para blindar nuestras bases de datos NoSQL, analizando autenticación RBAC, encriptación AES-256 en reposo, y prevención de inyecciones NoSQL.

---

[← Capítulo anterior (Capítulo 8)](08-consistencia-y-transacciones.md) | [Inicio (README.md)](README.md) | [Capítulo siguiente (Capítulo 10) →](10-seguridad.md)
