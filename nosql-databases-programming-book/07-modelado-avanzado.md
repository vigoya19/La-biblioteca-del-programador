# Capítulo 7: Modelado de Datos NoSQL Avanzado

> "En NoSQL, no desnormalizas porque seas perezoso para diseñar tablas; desnormalizas porque priorizas la velocidad física de lectura y la latencia del usuario final por encima de cualquier dogma académico."

Durante décadas, la normalización de datos (formas normales 1NF, 2NF, 3NF) dictó que cada fragmento de información debía residir en un único lugar físico del disco para evitar redundancias y garantizar la consistencia absoluta en la actualización. En la era relacional clásica, esto tenía sentido: el almacenamiento físico en disco era extremadamente costoso. 

Hoy, la economía del software ha cambiado por completo: **el espacio de disco duro es increíblemente barato, pero el tiempo de cómputo del procesador y la latencia de respuesta del usuario final son increíblemente caros**. En este capítulo, estudiaremos los patrones avanzados de modelado NoSQL, desde la desnormalización estratégica hasta las entrañas de **DynamoDB Single-Table Design** y los patrones de diseño de esquemas documentales más sofisticados del sector.

---

## 7.1 La Desnormalización Estratégica: El Intercambio del Futuro

La desnormalización es el proceso de duplicar de forma intencionada y controlada ciertos datos en múltiples registros para acelerar el tiempo de lectura física, eliminando las consultas JOIN en red.

```
┌──────────────────────────────────────────────┐
│  SQL NORMALIZADO (Minimiza Redundancia)      │ ──► Múltiples JOINs = Alta Latencia de Lectura.
└──────────────────────────────────────────────┘
┌──────────────────────────────────────────────┐
│  NoSQL DESNORMALIZADO (Minimiza Latencia)    │ ──► Lectura instantánea de un bloque.
└──────────────────────────────────────────────┘
```

* **El Trade-Off NoSQL**: Intercambias **complejidad de escritura** (cuando actualizas un dato, debes actualizarlo en múltiples registros duplicados) a cambio de **simplicidad y velocidad extrema de lectura** (obtienes el registro completo en una sola llamada de red).
* **Cuándo desnormalizar**: Ideal en sistemas donde la tasa de lectura supera drásticamente a la tasa de escritura (típicamente $95\%$ de lecturas frente a $5\%$ de escrituras, como catálogos, perfiles de usuario o feeds).

> [!NOTE]
> ### 📑 La Copia Fotostática de Seguridad en Cada Expediente
> 
> Visualicemos el modelado de datos en tu oficina:
> 
> - **El Enfoque Relacional SQL (Normalización)** es equivalente a una oficina hiper-organizada:
>   - Tienes el **Edificio de Identificaciones** (Tabla Usuarios), el **Edificio de Finanzas** (Tabla Facturas) y el **Edificio de Logística** (Tabla Direcciones).
>   - Cada vez que un cliente te pide procesar un pedido, debes enviar a un mensajero corriendo por la calle a buscar el acta de nacimiento al primer edificio, luego al segundo a buscar el historial de pago, y luego al tercero a verificar la dirección física (**múltiples JOINs**). No hay datos duplicados, pero tu mensajero tarda 1 hora por trámite.
> - **El Enfoque NoSQL (Desnormalización)** es equivalente a un archivador moderno:
>   - Cuando abres la carpeta del cliente (un **Documento de MongoDB** o una **Fila de DynamoDB**), encuentras una **copia fotostática (fotocopia)** de su documento de identidad grapada directamente al historial de facturas.
>   - El mensajero abre la carpeta y en 1 segundo tiene toda la información junta bajo su mano.
>   - **La penalización**: Si el cliente cambia su segundo nombre de "Andrés" a "Andy", tu secretaria tendrá que buscar y reescribir manualmente ese nombre en las copias fotostáticas de todas las carpetas del archivador (**consistencia eventual en escrituras**). Sin embargo, el 99% de los días el cliente no cambia su nombre; solo lee sus facturas. El ahorro de tiempo es monumental.

---

## 7.2 Patrones de Esquemas en MongoDB: Embeber vs. Referenciar

En MongoDB, la principal decisión de diseño es determinar cuándo almacenar datos anidados en un solo documento (**Embeber**) y cuándo enlazarlos en colecciones separadas mediante IDs (**Referenciar**).

```
   EMBEBER (Documentos Anidados)                   REFERENCIAR (Esquema por ID)
   ┌────────────────────────────────┐             ┌──────────────┐     ┌──────────────┐
   │ Documento: Usuario             │             │ Col: Usuario │     │ Col: Pedidos │
   │ - nombre: "Andy"               │             │ - ID: 1      │ ◄── │ - userId: 1  │
   │ - direcciones: [{calle: "x"}]  │             └──────────────┘     └──────────────┘
   └────────────────────────────────┘
```

### Reglas de Oro del Modelado Documental:
1. **Embeber si**:
   * Existe una relación de pertenencia exclusiva (1:1 o 1:pocos). Las direcciones pertenecen a un solo usuario y es raro consultarlas de forma aislada.
   * El número de elementos hijos está estrictamente acotado (**Bounded**). Por ejemplo, un usuario no tendrá más de 5 direcciones.
   * Los datos hijos se leen casi siempre al mismo tiempo que el padre.
2. **Referenciar si**:
   * Los elementos hijos crecen de forma ilimitada (**Unbounded**). Un sensor de IoT genera millones de registros por segundo. Si intentas embeberlos en el documento del sensor, colapsarás rápidamente el límite físico estricto de **16 MB por documento de MongoDB**.
   * Los elementos se comparten entre múltiples entidades (relación M:N). Un producto del catálogo pertenece a múltiples categorías dinámicas.

---

## 7.3 DynamoDB Single-Table Design (Diseño de Tabla Única)

En DynamoDB, el patrón de diseño más avanzado y de nivel arquitectónico sénior es el **Single-Table Design**. Consiste en almacenar **todas las entidades de negocio diferentes de tu dominio (Usuarios, Pedidos, Productos, Envíos) dentro de una única tabla física de base de datos**.

### ¿Cómo es posible mezclar peras con manzanas en la misma tabla?
1. Redefinimos los nombres de las claves de la tabla original con nombres genéricos: `PK` (Partition Key - String) y `SK` (Sort Key - String).
2. Utilizamos prefijos específicos en los strings de las claves para clasificar las entidades jerárquicamente en tiempo de ejecución.
3. Sobrecargamos los Índices Secundarios Globales (GSIs) utilizando claves genéricas `GSI1_PK` y `GSI1_SK`.

### Ejemplo Práctico de Estructura de Claves Unificada:

| Entity Type | Partition Key (PK) | Sort Key (SK) | Atributos Adicionales |
| :--- | :--- | :--- | :--- |
| **Usuario** | `USER#usr_100` | `METADATA` | `nombre: "Andrés"`, `email: "a@a.com"` |
| **Pedido** | `USER#usr_100` | `ORDER#ord_500` | `total: 150`, `fecha: "2026-05-22"` |
| **Pedido** | `USER#usr_100` | `ORDER#ord_501` | `total: 80`, `fecha: "2026-05-23"` |
| **Producto** | `PRODUCT#prod_90` | `METADATA` | `precio: 45`, `nombre: "Teclado"` |

* **Búsqueda Multipropósito de Alta Latencia Constante**:
  * Si ejecutas una Query donde `PK = "USER#usr_100"` y `SK = "METADATA"`, obtienes instantáneamente el perfil de Andrés.
  * Si ejecutas una Query donde `PK = "USER#usr_100"` y `SK begins_with("ORDER#")`, DynamoDB te devolverá de forma contigua y en un solo salto de disco **todos los pedidos históricos de Andrés en milisegundos**, sin realizar ningún JOIN relacional.

---

## 7.4 Patrones de Diseño Específicos

### 1. Bucket Pattern (Patrón de Balde)
Utilizado para manejar datos de series temporales de alta velocidad (como sensores IoT o logs). En lugar de crear un documento por cada registro individual del sensor (lo que infla los índices en disco), agrupamos las lecturas en bloques contiguos pre-asignados (por ejemplo, un documento por hora):
```json
{
  "sensorId": "sensor_10",
  "hora": "2026-05-22T21:00:00Z",
  "lecturas": [
    {"timestamp": "21:01", "temperatura": 24.5},
    {"timestamp": "21:02", "temperatura": 24.7}
  ]
}
```

### 2. Extended Reference Pattern (Referencia Extendida)
Si tienes una colección de `Pedidos` que requiere información del `Usuario`, en lugar de referenciar el ID del usuario y realizar un `$lookup` JOIN que ralentiza las lecturas, duplicamos los 2 o 3 campos más comúnmente leídos (como el `nombre` e `imagenAvatar`), manteniendo el resto del perfil pesado en la colección principal:
```json
{
  "_id": "pedido_500",
  "total": 150.00,
  "comprador": {
    "usuarioId": "usr_100",
    "nombre": "Andrés"
  }
}
```

---

## 7.5 Implementación en TypeScript de Modelos Avanzados

Implementemos el patrón de **Extended Reference** en MongoDB utilizando **Mongoose** en TypeScript:

#### [EsquemaPedido.ts](file:///Users/andres/Documents/biblioteca/nosql-databases-programming-book/src/models/EsquemaPedido.ts)
```typescript
import { Schema, model, Document } from 'mongoose';

// 1. Interfaz del usuario embebido de forma parcial (Extended Reference)
interface IUsuarioReferencia {
  usuarioId: Schema.Types.ObjectId;
  nombre: string;
  email: string;
}

// 2. Interfaz principal del pedido
export interface IPedido extends Document {
  total: number;
  productos: string[];
  cliente: IUsuarioReferencia; // Referencia extendida embebida
  fechaCompra: Date;
}

const UsuarioReferenciaSchema = new Schema<IUsuarioReferencia>({
  usuarioId: { type: Schema.Types.ObjectId, required: true },
  nombre: { type: String, required: true },
  email: { type: String, required: true }
}, { _id: false }); // Evitamos generar un _id para el subdocumento interno

const PedidoSchema = new Schema<IPedido>({
  total: { type: Number, required: true },
  productos: { type: [String], required: true },
  cliente: { type: UsuarioReferenciaSchema, required: true },
  fechaCompra: { type: Date, default: Date.now }
});

// 3. Crear el modelo físico
export const Pedido = model<IPedido>('Pedido', PedidoSchema);
```

---

## 7.6 Deep Dive: Listas de Adyacencia y Sobrecarga de GSIs

Cuando te enfrentas a arquitecturas enterprise que modelan cientos de entidades de negocio interconectadas en DynamoDB, el **Single-Table Design** básico se queda corto. Los ingenieros de sistemas sénior emplean dos patrones matemáticos de modelado extremadamente avanzados para superar las limitaciones del hardware en Cloud:

> [!NOTE]
> ### 📦 La Analogía de los Cajones Genéricos y los Ficheros de Membresía Duplicados
> 
> Visualicemos la sobrecarga de índices y las relaciones Muchos a Muchos (M:N) sin JOINs:
> 
> - **La Sobrecarga de Índices (GSI Overloading)** es equivalente a una **Restricción de Mobiliario en la Oficina**:
>   - El dueño del edificio te dice que solo puedes instalar un **Cajón Adicional con Llave (el GSI1)** en tu escritorio para emergencias.
>   - Si rotulas el cajón como *"Cajón para Correos"*, ya no puedes usarlo para facturas ni productos.
>   - **La Solución**: Dejas el cajón con una etiqueta en blanco (**`GSI1_PK`**).
>   - Si guardas una **Factura**, escribes arriba con un lápiz grueso: `"ESTADO#PENDIENTE"`.
>   - Si guardas un **Producto**, escribes arriba: `"CATEGORIA#TECNOLOGIA"`.
>   - Cuando abres el cajón, como todo está ordenado alfabéticamente, todas las facturas se agrupan juntas al inicio bajo la letra "E" de "ESTADO", y todos los productos bajo la "C" de "CATEGORIA". Has resuelto búsquedas de dos mundos completamente distintos usando el mismo mueble físico.
> - **La Lista de Adyacencia (M:N)** es equivalente a un **Club Escolar**:
>   - Tienes alumnos y talleres (un alumno va a muchos talleres, un taller tiene muchos alumnos).
>   - En lugar de tener una pizarra central de cruces relacionales que todos deben consultar haciendo cola, imprimes **Dos Fichas de Cartulina por cada inscripción**:
>     1. Metes una cartulina en la **Carpeta de Juan (el Alumno)** que dice: *"Taller: Ajedrez"*.
>     2. Metes la otra cartulina en la **Carpeta de Ajedrez (el Taller)** que dice: *"Alumno: Juan"*.
>   - Si quieres saber los talleres de Juan, abres su carpeta de un tirón. Si quieres saber los miembros de Ajedrez, abres su carpeta de un tirón. No hay cruces de habitaciones ni esperas. Todo está a un movimiento de mano.

### 1. Sobrecarga de Índices (GSI Overloading)
AWS DynamoDB impone un límite por defecto de **20 Índices Secundarios Globales (GSIs)** por cada tabla física. Si intentas crear un GSI dedicado para cada consulta individual de tu aplicación (como un índice para `Email`, otro para `EstadoPedido`, otro para `CategoriaProducto`), agotarás el límite de AWS de inmediato y pagarás una fortuna en duplicación de almacenamiento.


Para erradicar esto, aplicamos **GSI Overloading**:
* Diseñamos una única pareja de claves de índice secundario genéricas llamadas **`GSI1_PK`** y **`GSI1_SK`** durante la creación original de la tabla.
* Al escribir registros en la tabla, inyectamos strings completamente discrepantes en estos dos campos según la entidad:
  * Para una **Factura**: Escribimos `GSI1_PK = "ESTADO#PENDIENTE"` y `GSI1_SK = "FECHA#2026-05-22"`.
  * Para un **Producto**: Escribimos `GSI1_PK = "CATEGORIA#TECNOLOGIA"` y `GSI1_SK = "PRECIO#000450"`.
* **El Milagro Arquitectónico**: Con un único GSI genérico físico sobrecargado, puedes resolver patrones de acceso radicalmente diferentes: *Listar facturas pendientes por fecha* y *Listar productos de tecnología ordenados por precio*, optimizando los costos de AWS al 90%.

### 2. Listas de Adyacencia para Relaciones Muchos a Muchos (M:N)
En bases relacionales, una relación M:N (por ejemplo, *Usuarios* pertenecientes a múltiples *Grupos de Chat*) se resuelve mediante una tabla intermedia de JOINs. En DynamoDB no existen los JOINs. ¿Cómo representamos esto de forma eficiente?

Implementamos una **Lista de Adyacencia (Adjacency List)**:
* Escribimos dos copias de relaciones simétricas en la tabla única por cada emparejamiento físico:
  1. **El Registro de Membresía del Usuario**: `PK = "USER#usr_100"`, `SK = "GROUP#grp_5"`. (Almacena atributos específicos de este usuario en este grupo, como `fechaUnion`).
  2. **El Registro de Miembro del Grupo**: `PK = "GROUP#grp_5"`, `SK = "USER#usr_100"`. (Almacena metadatos del usuario legibles desde el grupo, como `nombreUsuario`).
* **Consultas de Tiempo Constante**:
  * *¿A qué grupos pertenece el usuario 100?*: Ejecutas una Query donde `PK = "USER#usr_100"` y `SK begins_with("GROUP#")`.
  * *¿Quiénes son todos los miembros del grupo 5?*: Ejecutas una Query donde `PK = "GROUP#grp_5"` y `SK begins_with("USER#")`.
* Ambas preguntas se resuelven en microsegundos en un solo salto de disco, emulando la adyacencia de grafos dentro de una tabla clave-valor estructurada.

---

## Resumen del Capítulo


* La **desnormalización estratégica** duplica datos controladamente para eliminar los costosos JOINs físicos en red, priorizando lecturas ultra-rápidas a cambio de escrituras complejas.
* En MongoDB, la regla cardinal para **Embeber** es la presencia de relaciones 1:N acotadas en tamaño; se **Referencia** cuando los datos crecen de forma ilimitada.
* **DynamoDB Single-Table Design** consolida múltiples entidades lógicas en una única tabla física mediante el diseño de llaves genéricas (`PK`/`SK`), logrando latencias constantes a escala extrema.
* Patrones como **Bucket** y **Extended Reference** optimizan el almacenamiento y previenen el desperdicio de índices y llamadas repetidas en red.

En el próximo capítulo, abordaremos uno de los retos más complejos del desarrollo de software: la **Consistencia Eventual** en sistemas distribuidos, analizando la resolución de conflictos (Vector Clocks), transacciones distribuidas y la orquestación mediante el patrón **Saga**.

---

[← Capítulo anterior (Capítulo 6)](06-neo4j.md) | [Inicio (README.md)](README.md) | [Capítulo siguiente (Capítulo 8) →](08-consistencia-y-transacciones.md)
