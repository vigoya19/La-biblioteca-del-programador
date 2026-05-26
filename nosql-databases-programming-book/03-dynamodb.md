# Capítulo 3: Amazon DynamoDB y el Almacenamiento Clave-Valor

> "En DynamoDB, no diseñas tu base de datos modelando las relaciones de tu negocio; la diseñas modelando de forma estricta los patrones de acceso de tus consultas."

En el diseño de sistemas distribuidos a escala de internet, muy pocos servicios de almacenamiento ofrecen el nivel de velocidad predecible y escalabilidad infinita que provee **Amazon DynamoDB**. Diseñada internamente por Amazon para dar soporte a sus sistemas de e-commerce durante los picos de tráfico de eventos como el Black Friday, DynamoDB es una base de datos NoSQL clave-valor e híbrida documental totalmente administrada.

El gran valor diferencial de DynamoDB es que **su latencia de respuesta sub-milisegundo es idéntica si tu tabla almacena 100 megabytes o 100 terabytes de datos**. En este capítulo, desmitificaremos su arquitectura distribuida, aprenderemos a modelar llaves primarias compuestas (**PK y SK**), diseñaremos índices secundarios (**GSIs y LSIs**), y aprenderemos a blindar el sistema contra particiones calientes (*hot partitions*) utilizando el SDK de AWS en TypeScript.

---

## 3.1 La Arquitectura Distribuida y el Hashing de Particiones

Para entender por qué DynamoDB es infinitamente escalable, debemos comprender cómo almacena físicamente tus datos. DynamoDB no guarda toda tu tabla en una sola máquina. En su lugar, fragmenta tus datos de forma transparente en múltiples bloques físicos de almacenamiento en disco llamados **Particiones**.

```
                           ┌───────────────────────────┐
                           │   Cliente (API Request)   │
                           └─────────────┬─────────────┘
                                         │  (Query: PK = "USER#100")
                                         ▼
                           ┌───────────────────────────┐
                           │   Algoritmo de Hashing    │  (MD5/MurmurHash)
                           └─────────────┬─────────────┘
                                         │  (Resultado Hash: 7b2a9...)
                                         ▼
┌───────────────────────────┬────────────┴──────────────┬───────────────────────────┐
│       Partición 1         │        Partición 2        │        Partición 3        │
│   (Rango Hash: 000-5ff)   │   (Rango Hash: 600-aff)   │   (Rango Hash: b00-fff)   │
│                           │  [ Aloja: USER#100 ]      │                           │
└───────────────────────────┴───────────────────────────┴───────────────────────────┘
```

Cuando realizas una petición de lectura o escritura especificando la llave de la partición, DynamoDB toma el valor de esa llave, lo pasa por una función de **hashing criptográfico** muy rápida (MurmurHash), obtiene un resultado numérico y sabe de forma instantánea a qué partición física dirigirse por cable directo, logrando accesos de tiempo constante:

$$O(1)$$

---

## 3.2 El Diseño de Llaves Primarias Compuestas: PK y SK

En DynamoDB, existen dos formas de definir la llave de identificación de tu tabla:

1.  **Llave Primaria Simple**: Compuesta únicamente por una **Partition Key (PK)**. Es un modelo clave-valor puro.
2.  **Llave Primaria Compuesta**: Compuesta por una **Partition Key (PK)** y una **Sort Key (SK)**. Este diseño habilita relaciones de uno a muchos (1:N) de forma nativa.

> [!NOTE]
> ### 🔑 La Analogía del Almacén de Casilleros y Estanterías Ordenadas
> 
> Imagina que eres el director del centro logístico de mensajería más grande del país:
> 
> - La **Partition Key (PK)** es equivalente al **Número exacto de casillero físico de seguridad** en la pared del almacén (por ejemplo, `USER#100`). Cuando llega una carta, el algoritmo de hashing te dice instantáneamente en qué casillero meterla. No tienes que rebuscar en todo el edificio; vas al casillero exacto en segundos, sin importar si el almacén tiene 10 o 1,000,000 de casilleros (**latencia predecible**).
> - La **Sort Key (SK)** es equivalente a las **carpetas ordenadas cronológicamente** dentro de ese casillero (por ejemplo, `PEDIDO#2026-05-22`). Permite que un cliente almacene múltiples expedientes relacionados en su propio casillero ordenados por fecha o alfabeto.
> - **El peligro del Scan (Filtro sin Llave)**: Si intentas hacer una búsqueda en tu base de datos sin especificar la Partition Key, es equivalente a ordenar a un operario que camine y abra de forma manual todos y cada uno de los millones de casilleros del almacén uno por uno para ver si el nombre coincide (**Scan**). Es una operación extremadamente lenta, que colapsa el almacén y te cuesta una fortuna de facturación.
> - **Hot Partitions (Particiones Calientes)**: Si envías 10,000 camiones a entregar paquetes en un solo casillero al mismo tiempo, el operario de ese casillero se agotará y el sistema colapsará (**Throttling**). Debes repartir las llaves de partición de forma homogénea por todo el almacén.

---

## 3.3 Índices Secundarios: GSIs y LSIs

¿Qué pasa si necesitas buscar un pedido no por el ID del usuario (tu PK), sino por el estado del envío o la fecha? Para esto DynamoDB provee dos tipos de índices secundarios:

*   **Local Secondary Index (LSI)**: Utiliza la **misma Partition Key** que la tabla original, pero una **Sort Key diferente**. Debe crearse obligatoriamente durante el andamiaje inicial de la tabla y comparte el límite de almacenamiento de 10 GB por partición.
*   **Global Secondary Index (GSI)**: Puede definir una **Partition Key y una Sort Key completamente diferentes** a las de la tabla original. Puede crearse y destruirse en cualquier momento, actúa como una tabla replicada de fondo y no tiene límites de tamaño de almacenamiento.

---

## 3.4 El SDK de AWS en TypeScript: Operaciones Atómicas y Condicionales

Para interactuar con DynamoDB de forma profesional en TypeScript, empleamos el SDK oficial de AWS (`@aws-sdk/client-dynamodb`):

### `dynamoClient.ts`
```typescript
import { DynamoDBClient } from '@aws-sdk/client-dynamodb';
import { DynamoDBDocumentClient, PutCommand, UpdateCommand } from '@aws-sdk/lib-dynamodb';

// 1. Inicializar el cliente básico de AWS
const clienteBase = new DynamoDBClient({ region: 'us-east-1' });

// 2. Envolver en el DocumentClient para simplificar el mapeo de tipos JSON de JS a Dynamo Types
export const dynamoDb = DynamoDBDocumentClient.from(clienteBase);
```

### Implementando una Escritura Condicional de Inventario:
Las escrituras condicionales son fundamentales para evitar colisiones en sistemas concurrentes (por ejemplo, evitar vender un producto si el stock llega a 0):

```typescript
import { UpdateCommand } from '@aws-sdk/lib-dynamodb';
import { dynamoDb } from './clients/dynamoClient';

interface ResultadoCompra {
  success: boolean;
  message: string;
}

async function procesarCompraDeStock(productoId: string, cantidadAComprar: number): Promise<ResultadoCompra> {
  const comando = new UpdateCommand({
    TableName: 'TablaProductos',
    Key: {
      PK: `PRODUCTO#${productoId}`,
      SK: 'METADATA'
    },
    // Expresión de actualización matemática atómica
    UpdateExpression: 'SET stock = stock - :cantidad',
    // Expresión condicional protectora: solo ejecuta si el stock es suficiente
    ConditionExpression: 'stock >= :cantidad',
    ExpressionAttributeValues: {
      ':cantidad': cantidadAComprar
    }
  });

  try {
    await dynamoDb.send(comando);
    return { success: true, message: 'Inventario descontado con éxito.' };
  } catch (error: any) {
    if (error.name === 'ConditionalCheckFailedException') {
      return { success: false, message: 'Error: Stock insuficiente para procesar la transacción.' };
    }
    throw error;
  }
}
```

---

## Resumen del Capítulo

*   Amazon DynamoDB garantiza **latencias constantes sub-milisegundo** gracias a su arquitectura distribuida basada en el hashing de particiones físicas.
*   El diseño compuesto de **Partition Key (PK)** y **Sort Key (SK)** permite modelar de forma predeterminada relaciones de tipo uno a muchos (1:N).
*   Los **GSIs** flexibilizan el modelo permitiendo reordenar y consultar los datos bajo llaves completamente diferentes de forma replicada asíncrona.
*   Las **escrituras condicionales** e incremento atómico en el SDK de AWS son la primera línea de defensa para prevenir carreras de concurrencia e inconsistencias en inventarios sin bloquear hilos.

En el próximo capítulo, entraremos de lleno al universo del almacenamiento en memoria de velocidad extrema, analizando en detalle **Redis**, sus estructuras complejas y las opciones de persistencia RDB y AOF en producción.

---

[Capítulo anterior](02-mongodb.md) | [Inicio](README.md) | [Capítulo siguiente →](04-redis.md)
