# Capítulo 2: MongoDB y el Almacenamiento Orientado a Documentos

> "La flexibilidad de un esquema documental no significa ausencia de estructura; significa que la estructura se adapta a las necesidades del negocio de forma orgánica, no a la inversa."

En las bases de datos relacionales, el modelado de datos nos obliga a fragmentar nuestras entidades en múltiples tablas normalizadas mediante claves foráneas. Si necesitas consultar el perfil de un usuario junto a sus direcciones, compras y métodos de pago, el motor SQL debe realizar múltiples uniones cruzadas (**JOINs**), lo que incrementa el consumo de memoria de disco y penaliza el rendimiento.

**MongoDB** rompe este esquema al proponer el **modelo orientado a documentos**. Permite almacenar información compleja, jerárquica y anidada en un solo registro autónomo. En este capítulo, estudiaremos el paso de JSON a almacenamiento binario BSON, la inyección y manipulación de datos en TypeScript mediante Mongoose, el potente **Aggregation Framework** y la infraestructura de clusters (Replica Sets y Sharding).

---

## 2.1 El Modelo Documental: ¿Por qué es Revolucionario?

MongoDB organiza la información en **Colecciones** (equivalentes a las tablas en SQL) que contienen **Documentos** (equivalentes a las filas). Sin embargo, a diferencia de las filas de SQL, los documentos de MongoDB son auto-descriptivos y con **esquema dinámico (schema-less)**.

> [!NOTE]
> ### 📁 La Analogía del Archivador de Carpetas de Manila vs. La Hoja de Cálculo Rígida
> 
> Imagina que estás administrando el registro de expedientes de clientes de una corporación:
> 
> - **El Enfoque SQL** es equivalente a usar una **Hoja de Cálculo de Excel Rígida**: Cada fila representa un cliente, y todas las filas están obligadas a tener exactamente las mismas columnas. Si necesitas guardar un campo especial (como "nombre del cónyuge") para solo un cliente, debes alterar la estructura de toda la hoja, insertando una celda vacía (`NULL`) para los 10,000 clientes restantes de la empresa (**migración de esquema**).
> - **El Enfoque MongoDB** es equivalente a un **Archivador de Carpetas de Manila Físicas**: Cada cliente tiene su propia carpeta de cartón (**un Documento**).
> - Abres la carpeta del cliente A y encuentras un contrato de 20 páginas grapado junto a 3 números telefónicos.
> - Abres la carpeta del cliente B y solo hay una nota adhesiva de Post-It con una dirección escrita a lápiz.
> - El archivador de cartón (la **Colección**) no impone reglas de celdas vacías; simplemente clasifica las carpetas de forma dinámica. Puedes enriquecer el expediente de cualquier cliente al instante sin tener que reestructurar las carpetas de todos los demás.

---

## 2.2 Internals: De JSON a BSON (Binary JSON)

Aunque los desarrolladores escribimos y leemos datos en MongoDB utilizando sintaxis JSON, el motor de MongoDB almacena los datos en disco en un formato binario optimizado llamado **BSON (Binary JSON)**.

```
┌─────────────────────────┐               ┌─────────────────────────┐
│     JSON (Texto Plano)  │               │      BSON (Binario)     │
│ - Lectura humana fácil. │  ───────────► │ - Tipado estricto (date)│
│ - Lento de parsear.     │    (Build)    │ - Saltos por byte offset│
│ - Sin tipos avanzados.  │               │ - Ultrarrápido en disco │
└─────────────────────────┘               └─────────────────────────┘
```

### ¿Por qué BSON y no JSON?
1.  **Tipado Estricto**: JSON tiene tipos de datos limitados (no distingue enteros de flotantes, ni tiene un tipo nativo para fechas). BSON añade soporte de primer nivel para `Int32`, `Int64`, `Double`, `Date` y `Binary Data` (UUIDs, imágenes).
2.  **Optimización de Búsqueda (Byte Offsets)**: En un archivo JSON de texto plano, para encontrar el campo `"edad": 30` al final del archivo, el motor debe leer y parsear carácter por carácter cada elemento anterior. BSON almacena al inicio del documento el tamaño de cada campo en bytes; esto le permite al motor **saltar físicamente en el disco** directo al byte exacto del campo buscado, logrando lecturas a velocidad de vértigo.

---

## 2.3 Conectando MongoDB con Mongoose y TypeScript

En el ecosistema de Node.js/TypeScript, **Mongoose** actúa como el ODM (Object Document Mapper) estándar para estructurar esquemas y dotar a nuestro código de validaciones y tipado estricto:

### `UsuarioModel.ts`
```typescript
import { Schema, model, Document } from 'mongoose';

// 1. Declarar la interfaz estricta de TypeScript
export interface IUsuario extends Document {
  nombre: string;
  email: string;
  edad: number;
  roles: string[];
  activo: boolean;
  fechaCreacion: Date;
}

// 2. Definir el esquema físico de validación de Mongoose
const UsuarioSchema = new Schema<IUsuario>({
  nombre: { type: String, required: true, minlength: 3 },
  email: { type: String, required: true, unique: true, index: true },
  edad: { type: Number, required: true, min: 18 },
  roles: { type: [String], default: ['user'] },
  activo: { type: Boolean, default: true },
  fechaCreacion: { type: Date, default: Date.now }
});

// 3. Crear y exportar el modelo fuertemente tipado
export const Usuario = model<IUsuario>('Usuario', UsuarioSchema);
```

---

## 2.4 Aggregation Framework: Canalizaciones Avanzadas

El **Aggregation Framework** de MongoDB es un motor de procesamiento de datos de altísimo rendimiento basado en el concepto de **tubería o canalización (Pipeline)**. El pipeline toma una colección de documentos, los pasa a través de una serie de etapas secuenciales (**Stages**) donde los documentos se filtran, agrupan, transforman y enriquecen para escupir un resultado final agregado en una sola consulta de red.

```
       [ Colección de Entrada ]
                  │
                  ▼
         ┌─────────────────┐
         │ 1. Stage $match │ ◄── [ Filtra usuarios mayores de 25 años ]
         └────────┬────────┘
                  │
                  ▼
         ┌─────────────────┐
         │ 2. Stage $group │ ◄── [ Agrupa por rol y calcula la edad promedio ]
         └────────┬────────┘
                  │
                  ▼
         ┌─────────────────┐
         │ 3. Stage $sort  │ ◄── [ Ordena descendente por edad promedio ]
         └────────┬────────┘
                  │
                  ▼
         [ Documentos de Salida ]
```

### Ejemplo Práctico: Pipeline de Métricas Corporativas

Implementemos una consulta analítica para agrupar usuarios activos, calcular su edad promedio y filtrar solo los roles que superen un umbral mínimo en TypeScript:

```typescript
import { Usuario } from './models/UsuarioModel';

async function obtenerMetricasDeRoles(): Promise<any[]> {
  const pipeline = [
    // Etapa 1: Filtrar solo usuarios activos
    { $match: { activo: true } },
    
    // Etapa 2: Agrupar por rol, contar miembros y calcular edad promedio
    {
      $group: {
        _id: '$roles', // Campo por el cual agrupar (desglosa el array)
        totalMiembros: { $sum: 1 },
        edadPromedio: { $avg: '$edad' }
      }
    },
    
    // Etapa 3: Filtrar solo grupos que tengan más de 2 miembros
    { $match: { totalMiembros: { $gt: 2 } } },
    
    // Etapa 4: Ordenar de forma descendente por edad promedio
    { $sort: { edadPromedio: -1 } }
  ];

  // Ejecutar el pipeline en el motor de MongoDB
  const resultados = await Usuario.aggregate(pipeline);
  return resultados;
}
```

---

## 2.5 Alta Disponibilidad y Escalabilidad: Replica Sets y Sharding

Para aplicaciones enterprise, MongoDB ofrece dos mecanismos de clusters distribuidos:

### 1. Replica Sets (Alta Disponibilidad y Resiliencia)
Un **Replica Set** es un grupo de instancias de MongoDB que mantienen exactamente el mismo conjunto de datos.
*   **Nodo Primario**: El único que recibe las operaciones de escritura.
*   **Nodos Secundarios**: Copian de forma asíncrona el registro de operaciones del primario para replicar los datos. Si el nodo primario se cae, los secundarios inician un **proceso de elecciones automático** para votar y elegir a un nuevo primario en milisegundos, garantizando tolerancia a fallos.

### 2. Sharding (Escalabilidad Horizontal Masiva)
Cuando el volumen de datos supera el espacio físico de disco de una sola máquina, se implementa **Sharding**:
*   Consiste en fragmentar y distribuir la colección a través de múltiples servidores independientes (**Shards**).
*   Un enrutador de consultas (**Mongos**) intercepta la petición del cliente y la dirige de forma directa al shard exacto que almacena el trozo de datos buscado, logrando escrituras concurrentes ilimitadas.

---

## Resumen del Capítulo

*   MongoDB se basa en el **modelo documental flexible (BSON)**, que permite almacenar estructuras dinámicas complejas y embebidas sin requerir un esquema rígido.
*   **BSON es un almacenamiento binario tipado** que permite saltar por byte offsets en disco, agilizando enormemente el tiempo de lectura frente a JSON tradicional.
*   El **Aggregation Framework** opera como una tubería secuencial de alto rendimiento para procesar, filtrar y agrupar millones de documentos directamente en la base de datos.
*   La resiliencia se garantiza mediante **Replica Sets** (elección de primario automática), mientras que la escalabilidad horizontal se logra particionando datos con **Sharding**.

En el próximo capítulo, estudiaremos el almacenamiento Clave-Valor de alto rendimiento, analizando los internals de **Amazon DynamoDB**, el diseño de llaves compuestas PK/SK y las GSIs.

---

[Capítulo anterior](01-introduccion-y-teorema-cap.md) | [Inicio](README.md) | [Capítulo siguiente →](03-dynamodb.md)
