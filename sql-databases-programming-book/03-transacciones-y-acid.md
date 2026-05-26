# Capítulo 3: Transacciones y Garantías ACID Estrictas

> "Una transacción no es un bloque de código agrupado por comodidad estética; es una garantía de inmutabilidad atómica frente al caos inevitable de la física del hardware, los cortes de energía eléctrica y las colisiones en red."

La mayor ventaja competitiva y el pilar fundamental que ha mantenido a las bases de datos relacionales en la cúspide de la confiabilidad del software es su capacidad para garantizar la integridad absoluta de tu negocio mediante transacciones **ACID**. En sistemas financieros, e-commerce, o gestores de inventarios, permitir que un fallo intermedio corrompa los datos o deje registros a medias es inaceptable.

En este capítulo, deconstruiremos microscópicamente las garantías ACID, estudiaremos el motor físico de durabilidad de PostgreSQL mediante el **Write-Ahead Logging (WAL)** y los hilos de **Checkpoints**, y aprenderemos a estructurar transacciones seguras con control estricto de excepciones en TypeScript.

---

## 3.1 Deconstruyendo ACID Microscópicamente

El acrónimo **ACID** define cuatro propiedades matemáticas fundamentales que el motor de base de datos relacional se compromete a cumplir de forma estricta:

1. **Atomicidad (Atomicity)**: La regla del **"todo o nada"**. Una transacción puede contener decenas de operaciones individuales (`INSERT`, `UPDATE`, `DELETE`). Si todas tienen éxito, la transacción se consolida en disco (**Commit**). Si una sola de ellas falla (por ejemplo, se corta el cable de red o se viola una restricción de clave única), el motor revierte de forma automática y absoluta todas las operaciones anteriores en disco (**Rollback**), dejando la base de datos como si nada hubiera ocurrido.
2. **Consistencia (Consistency)**: La base de datos garantiza que una transacción solo puede hacer la transición del sistema de un estado de esquema válido a otro. Si intentas transferir dinero a una cuenta que no existe o insertar un valor negativo en un campo marcado con una restricción `CHECK (precio > 0)`, el motor abortará la transacción para impedir que se violen las reglas lógicas del negocio.
3. **Aislamiento (Isolation)**: Garantiza que la ejecución concurrente de múltiples transacciones deje a la base de datos en un estado físico idéntico a como si las transacciones se hubieran ejecutado de forma secuencial una detrás de otra. Evita que transacciones concurrentes lean o modifiquen datos intermedios "sucios" de otras transacciones aún no consolidadas.
4. **Durabilidad (Durability)**: Garantiza que, una vez que la base de datos confirma al cliente que una transacción ha sido consolidada con éxito (`Commit`), los cambios persistirán de forma permanente en el almacenamiento secundario físico (SSD/HDD), resistiendo apagones repentinos del servidor o colapsos del sistema operativo.

---

## 3.2 Internals de Durabilidad Física: WAL y Checkpoints

¿Cómo logra un motor como PostgreSQL garantizar que los datos estén seguros en el disco físico sin ralentizar de forma catastrófica las escrituras de tu aplicación?

Si cada vez que confirmas una transacción (un `Commit`), el motor tuviera que saltar al disco duro a buscar y modificar en caliente las páginas de datos distribuidas aleatoriamente por los archivos en disco duro (**Random I/O**), tu base de datos colapsaría a una velocidad de apenas unas pocas decenas de escrituras por segundo.

PostgreSQL resuelve esto implementando la arquitectura **Write-Ahead Logging (WAL)**:

```
[ Petición de Commit ]
          │
          ├──► [ 1. Write-Ahead Log (WAL) ] (Escritura física secuencial ultra-rápida)
          │
          └──► [ 2. Shared Buffers (RAM) ]   (Marca la página de datos como "Dirty Page")
                   │
                   ▼ (Periódicamente / En calma)
             [ 3. Checkpoint Thread ]        (Vuelca las "Dirty Pages" de la RAM al disco)
```

1. **El Log Secuencial (WAL) y el LSN (Log Sequence Number)**: Cuando ejecutas una escritura, PostgreSQL traduce la operación física a un registro WAL y le asigna un identificador único incremental de 64 bits llamado **LSN (Log Sequence Number)** (ej. `0/19A2E38`). Este LSN representa la posición física en bytes del registro dentro del log secuencial en disco.
2. **El Secreto de la Consistencia: `pd_lsn`**: Cada página de 8KB en los Shared Buffers de RAM contiene una cabecera con el campo **`pd_lsn`**. Cuando una página se modifica, se graba en su `pd_lsn` el LSN del registro WAL correspondiente. Durante el volcado asíncrono o la recuperación de catástrofes, el motor compara el LSN del registro WAL con el `pd_lsn` de la página en disco: **si el `pd_lsn` de la página es igual o mayor que el LSN del registro, el motor sabe que la página ya está escrita y omite el procesamiento redundante**, garantizando consistencia absoluta.
3. **fsync en el WAL**: En cuanto el log del WAL se graba y consolida en el disco físico mediante la llamada al sistema de sincronización de archivos de Unix (`fsync`), PostgreSQL le responde de inmediato "Éxito" al cliente. Los datos están 100% seguros y son durables.
4. **Shared Buffers (RAM)**: Al mismo tiempo, la modificación se realiza en caliente en la memoria RAM compartida de PostgreSQL, marcando esa página de memoria como una **Dirty Page (Página Sucia)**.
5. **Checkpoints (El Vuelco Asíncrono)**: Un hilo secundario de fondo de PostgreSQL llamado **Checkpoint Thread** se ejecuta periódicamente de forma asíncrona. Su misión es escanear los Shared Buffers en RAM, recoger todas las "Dirty Pages" acumuladas y volcarlas de forma secuencial y ordenada a los archivos de datos principales en el disco. Una vez completado, limpia los logs obsoletos del WAL.
6. **El Desastre de las Torn Pages (Páginas Rotas)**: Las páginas de PostgreSQL ocupan 8KB, pero la gran mayoría de los sistemas operativos (Unix/Linux) escriben bloques físicos a nivel de disco de 4KB. Si el servidor sufre un corte de energía eléctrica justo en el milisegundo en que PostgreSQL está volcando una página de 8KB, el sistema operativo podría escribir únicamente los primeros 4KB de la página dejando los otros 4KB corruptos e ilegibles. Esto se llama **Torn Page**.
7. **La Salvaguarda: Full Page Writes**: Para protegerse de las páginas rotas, PostgreSQL implementa **Full Page Writes**. Tras cada Checkpoint, **la primera vez que se modifica una página física de 8KB en memoria, PostgreSQL no graba solo el pequeño registro del cambio en el WAL; graba la página física de 8KB completa en el archivo WAL**. Si ocurre una caída y la página en el disco principal se rompe, el proceso de Crash Recovery detecta el fallo, recupera el bloque completo de 8KB intacto desde el WAL y luego aplica de forma segura los cambios subsecuentes.
8. **Crash Recovery (Recuperación ante fallos)**: Si el servidor sufre un apagón repentino en medio del vuelo y la RAM se vacía, al reiniciar, PostgreSQL detecta la caída, lee el archivo WAL secuencialmente desde el último checkpoint y vuelve a aplicar todas las operaciones de forma exacta (**Redo / Replay**), reconstruyendo la base de datos al milisegundo anterior a la catástrofe.

---

## 3.3 Afinación Avanzada de Durabilidad en Producción

En sistemas relacionales empresariales, podemos calibrar la velocidad física y durabilidad de transacciones mediante variables en `postgresql.conf`:

* **`synchronous_commit`** (Por defecto = `on`):
  * Si está en `on`, el comando `COMMIT` espera a que el WAL se escriba físicamente en disco (`fsync`) antes de responder éxito al usuario.
  * Si se configura en `off` (Replicación Asíncrona local), el motor responde de inmediato al cliente tras escribir en el buffer de RAM y flush el WAL a disco en segundo plano cada 0.5 segundos. **Multiplica por 10 la velocidad de escrituras del sistema**, a cambio de un riesgo menor de perder los últimos 0.5 segundos de datos si ocurre un apagón repentino (nunca corrompe la estructura de la base de datos).
* **`wal_buffers`** (Por defecto = `auto`, aprox. $3\%$ de `shared_buffers`): El tamaño de memoria dedicada para retener los registros del WAL en RAM antes de volcarlos a disco.
* **`checkpoint_completion_target`** (Por defecto = `0.9`): Indica que el hilo del Checkpoint debe intentar esparcir la escritura de las Dirty Pages a disco a lo largo del $90\%$ del tiempo asignado entre checkpoints, previniendo picos brutales de I/O en disco que congelen las consultas de red de los usuarios concurrentes.

> [!NOTE]
> ### ✈️ La Caja Negra del Avión y la Bitácora de Vuelo
> 
> Visualicemos el dilema de la durabilidad relacional:
> 
> - **El Enfoque Ineficiente (Escritura Directa a Disco)**:
>   - Imagina que eres el piloto de un avión comercial en medio de una tormenta de alta turbulencia. Cada vez que un pasajero te pide cambiar de asiento, detienes el avión en el aire, caminas a la cabina de pasajeros, mueves las maletas físicamente entre los compartimentos de equipaje y reordenas las cabinas (**Random I/O lento**). Tu vuelo colapsará por ineficiencia física.
> - **El Enfoque WAL (Write-Ahead Log)**:
>   - En la cabina de pilotaje tienes una **Bitácora / Caja Negra Blindada de Metal (el WAL)**.
>   - Cuando un pasajero te pide cambiar del asiento 2B al 5C, simplemente tomas un bolígrafo y anotas un renglón en la bitácora: *"Pasajero Juan se movió del 2B al 5C"*. Cierras la caja negra bajo llave. Al instante le dices al pasajero: *"Cambio realizado con éxito"*. Tardaste 0.1 segundos.
>   - **El Checkpoint**: Una azafata (el **Checkpoint Thread**) pasa periódicamente por la cabina del piloto cuando el vuelo está en calma, lee la bitácora y se encarga de mover físicamente las pesadas maletas en la parte de atrás de fondo de forma asíncrona.
>   - **Crash Recovery (Recuperación de Catástrofe)**: Si cae un rayo y el avión pierde la luz eléctrica por completo (un apagón del servidor), la azafata olvidará temporalmente dónde iba cada equipaje. Al encender las luces, el piloto simplemente abre la Caja Negra de metal inquebrantable, lee las anotaciones en orden cronológico y reubica las maletas exactamente en sus asientos finales. Ninguna maleta se pierde.

---

## 3.3 Implementación de Transacciones Robustas en TypeScript

Implementemos una transacción bancaria robusta en TypeScript con control de excepciones y rollback automático utilizando nuestro pool de conexiones de PostgreSQL:

### `transferenciaService.ts`
```typescript
import { dbPool } from '../clients/dbClient';

interface ResultadoTransferencia {
  success: boolean;
  message: string;
}

export async function procesarTransferenciaFinanciera(
  emisorId: number, 
  receptorId: number, 
  monto: number
): Promise<ResultadoTransferencia> {
  
  // 1. Adquirir una conexión dedicada del Pool para la transacción
  const cliente = await dbPool.connect();

  try {
    // 2. Iniciar formalmente el bloque transaccional en PostgreSQL
    await cliente.query('BEGIN');

    // 3. Paso A: Restar saldo al emisor de forma segura
    const queryRestar = `
      UPDATE cuentas 
      SET saldo = saldo - $1 
      WHERE usuario_id = $2 AND saldo >= $1
      RETURNING saldo
    `;
    const resultadoRestar = await cliente.query(queryRestar, [monto, emisorId]);

    // Validación lógica: Si no devolvió filas es porque no tenía saldo suficiente
    if (resultadoRestar.rows.length === 0) {
      throw new Error('Fondos insuficientes para procesar la transacción.');
    }

    // 4. Paso B: Sumar saldo al receptor
    const querySumar = `
      UPDATE cuentas 
      SET saldo = saldo + $1 
      WHERE usuario_id = $2
    `;
    const resultadoSumar = await cliente.query(querySumar, [monto, receptorId]);

    if (resultadoSumar.rowCount === 0) {
      throw new Error('La cuenta del receptor no existe o está inactiva.');
    }

    // 5. Consolidar de forma atómica y permanente en el WAL y disco
    await cliente.query('COMMIT');
    return { success: true, message: 'Transferencia completada con éxito.' };

  } catch (error: any) {
    // 6. Compensación: Revertir absolutamente cualquier cambio si algo falló
    await cliente.query('ROLLBACK');
    console.error('[Transacción-Error] Fallo en la transferencia bancaria:', error.message);
    return { success: false, message: `Error: ${error.message}` };

  } finally {
    // 7. Liberar la conexión para que vuelva al Pool de sockets reutilizables
    cliente.release();
  }
}
```

---

## Resumen del Capítulo

* Las garantías **ACID** son el pilar de consistencia relacional, asegurando Atomicidad (todo o nada), Consistencia (reglas de esquema), Aislamiento (concurrencia segura) y Durabilidad (permanencia física).
* La durabilidad de alto rendimiento se logra con la arquitectura **Write-Ahead Logging (WAL)**: las escrituras se registran de forma puramente secuencial rápida en disco físico antes de considerarse consolidadas.
* El hilo asíncrono **Checkpoint** vuelca periódicamente las páginas sucias de la memoria RAM compartida a los archivos de datos principales, aliviando la saturación de espacio del WAL.
* Al reiniciar ante caídas, el motor relacional lee el WAL para realizar una operación **Redo**, garantizando la integridad absoluta de la información en segundos.

En el próximo capítulo, ingresaremos a un área sumamente sofisticada del desarrollo relacional: los **Niveles de Aislamiento y Anomalías de Concurrencia**, desmitificando el funcionamiento interno de **MVCC** y previniendo el peligroso **Write Skew**.

---

[← Capítulo anterior (Capítulo 2)](02-anatomia-indexacion.md) | [Inicio (README.md)](README.md) | [Capítulo siguiente (Capítulo 4) →](04-niveles-aislamiento-y-mvcc.md)
