# Capítulo 5: Mecanismos de Bloqueos (Locking)

> "En un sistema altamente concurrente, la ausencia de bloqueos genera corrupción de datos; el exceso de bloqueos paraliza la infraestructura. El secreto del alto rendimiento radica en comprender la física de la contención."

Cuando múltiples usuarios intentan leer y escribir simultáneamente sobre los mismos registros en una base de datos relacional, el aislamiento MVCC (que estudiamos en el capítulo anterior) nos protege de anomalías de visibilidad. Sin embargo, cuando dos procesos intentan modificar el **mismo** registro exacto al mismo tiempo, el motor debe decidir cómo resolver este conflicto físico.

Aquí es donde entran los **Mecanismos de Bloqueos (Locking)**. PostgreSQL utiliza un sistema jerárquico de bloqueos automáticos para proteger la integridad de los datos, pero también nos proporciona herramientas explícitas para controlar la contención. En este capítulo, desglosaremos los bloqueos a nivel de fila y de tabla, el comportamiento de los deadlocks, las estrategias pesimistas vs. optimistas, y cómo escribir código TypeScript ultra seguro que evite la parálisis del sistema.

---

## 5.1 Bloqueos a Nivel de Fila (Row-Level Locks)

Cuando ejecutas una sentencia de modificación como `UPDATE usuarios SET saldo = 100 WHERE id = 5;`, PostgreSQL bloquea automáticamente esa fila de forma implícita. Si otra transacción concurrente intenta hacer un `UPDATE` sobre esa misma fila `id = 5`, será puesta en cola en un estado de espera hasta que la primera transacción ejecute un `COMMIT` o un `ROLLBACK`.

Existen modos de bloqueo de fila explícitos que podemos solicitar a través de sentencias `SELECT`:

### 1. `SELECT ... FOR UPDATE`
* **Comportamiento**: Adquiere un bloqueo exclusivo sobre las filas seleccionadas. Evita que otras transacciones las modifiquen, las eliminen o adquieran cualquier bloqueo sobre ellas (`FOR UPDATE`, `FOR NO KEY UPDATE`, `FOR SHARE`, `FOR KEY SHARE`).
* **Caso de uso**: Ideal cuando vas a leer una fila con el propósito explícito de actualizarla inmediatamente después en la misma transacción (por ejemplo, descontar saldo tras validar stock).

### 2. `SELECT ... FOR SHARE`
* **Comportamiento**: Adquiere un bloqueo compartido. Permite que otras transacciones lean y adquieran bloqueos compartidos sobre las mismas filas, pero **bloquea cualquier intento de modificación (`UPDATE`/`DELETE`) o bloqueo exclusivo** sobre ellas hasta que termine la transacción.
* **Caso de uso**: Perfecto para validar relaciones de integridad (por ejemplo, asegurar que un registro padre no sea eliminado mientras estás insertando un registro hijo relacionado).

### 3. Modificadores de Bloqueo Clave: `NOWAIT` y `SKIP LOCKED`
Por defecto, si una fila ya está bloqueada por otra transacción, tu consulta se colgará esperando indefinidamente. Podemos alterar esto:
* **`FOR UPDATE NOWAIT`**: Si la fila está bloqueada, la base de datos arroja inmediatamente un error de serialización (`55P03: lock_not_available`) en lugar de esperar. Esto permite al cliente abortar rápido y tomar rutas alternativas.
* **`FOR UPDATE SKIP LOCKED`**: Ignora silenciosamente las filas que ya estén bloqueadas por otras transacciones y devuelve únicamente las filas libres. **Es el motor fundamental para construir colas de mensajería o procesamiento de tareas concurrentes de alto rendimiento en SQL.**

---

## 5.2 Bloqueos a Nivel de Tabla (Table-Level Locks)

A diferencia de los bloqueos de fila, los bloqueos de tabla impiden operaciones a gran escala sobre la estructura de la base de datos (como alterar columnas o truncar registros). PostgreSQL gestiona **8 niveles de bloqueos de tabla** con una matriz de compatibilidad rigurosa en el núcleo del motor:

### Matriz de los 8 Modos de Bloqueo de Tabla:

1. **`ACCESS SHARE`**: Adquirido de forma automática por sentencias de pura lectura (`SELECT`).
2. **`ROW SHARE`**: Adquirido por `SELECT ... FOR UPDATE` y `FOR SHARE`.
3. **`ROW EXCLUSIVE`**: Adquirido por sentencias de modificación de datos (`UPDATE`, `DELETE`, `INSERT`).
4. **`SHARE UPDATE EXCLUSIVE`**: Adquirido por comandos de mantenimiento en caliente como `VACUUM` (ordinario), `ANALYZE`, `CREATE INDEX CONCURRENTLY` y `VALIDATE CONSTRAINT`.
5. **`SHARE`**: Adquirido por `CREATE INDEX` (estándar no concurrente).
6. **`SHARE ROW EXCLUSIVE`**: Bloquea modificaciones y exclusividades concurrentes en caliente.
7. **`EXCLUSIVE`**: Permite lecturas (`SELECT`) concurrentes pero bloquea absolutamente todo lo demás.
8. **`ACCESS EXCLUSIVE`**: El candado definitivo. Adquirido por comandos DDL como `ALTER TABLE`, `DROP TABLE`, `TRUNCATE` y `VACUUM FULL`. **Bloquea el $100\%$ de las lecturas y escrituras concurrentes**, provocando encolamientos y parálisis total si se retiene durante transacciones largas.

---

## 5.3 Advisory Locks (Bloqueos Lógicos Personalizados)

A menudo, las aplicaciones distribuidas necesitan coordinar una exclusión mutua para procesos de software lógicos (por ejemplo, asegurar que solo un worker ejecute la generación de un PDF pesado o evitar disparar llamadas duplicadas a APIs externas de pago de terceros). 

En lugar de instalar y configurar tecnologías pesadas externas como Redis (con Redlock) o ZooKeeper, PostgreSQL nos ofrece **Advisory Locks** nativos:
* **Mecánica**: Son bloqueos puramente lógicos creados a nivel del motor en base a números de 64 bits (`BIGINT`) definidos por la aplicación. No bloquean ninguna fila física de ninguna tabla real; solo bloquean la llave del candado numérico en la memoria RAM del motor.
* **Tipos de Alcance**:
  1. **Session-Level (Nivel de Sesión)**: El candado se retiene a lo largo de toda la conexión física de red. Se adquiere con `pg_advisory_lock(key)` y debe liberarse explícitamente llamando a `pg_advisory_unlock(key)`. Si la conexión se cae, PostgreSQL limpia el candado automáticamente.
  2. **Transaction-Level (Nivel de Transacción)**: El candado se asocia a la transacción en curso. Se adquiere mediante `pg_advisory_xact_lock(key)` y **se libera de forma automática y transparente en el momento del `COMMIT` o `ROLLBACK`**, evitando fugas de bloqueos lógicos.
* **Evitar esperas**: Al igual que `NOWAIT` en filas, podemos usar `pg_try_advisory_lock(key)` o `pg_try_advisory_xact_lock(key)`, que devuelven un booleano inmediato (`true`/`false`) si el candado estaba libre o ya estaba retenido por otro servidor del clúster de aplicaciones.

---

> [!NOTE]
> ### 🔐 Los Candados del Hotel y los Huéspedes Hambrientos
> 
> Entendamos cómo funcionan los bloqueos y los deadlocks mediante una clara analogía física:
> 
> - **El Bloqueo Compartido (`FOR SHARE` - El Mapa del Hotel)**:
>   - Imagina que un grupo de turistas llega al lobby de un hotel y pide ver el mapa físico del edificio que está colgado en la pared.
>   - Diez turistas pueden mirar el mapa concurrentemente sin ningún problema (**Bloqueo de Lectura Compartido**). Nadie estorba al otro porque solo están consumiendo la información.
>   - Sin embargo, si un pintor del hotel llega con una brocha para remodelar la pared donde está el mapa (**Modificación/Bloqueo Exclusivo**), el conserje le impedirá hacerlo hasta que todos los turistas terminen de mirar y se retiren del lobby.
> 
> - **El Bloqueo Exclusivo (`FOR UPDATE` - El Baño de la Habitación)**:
>   - Un huésped entra al baño de su habitación y echa el cerrojo por dentro (**Bloqueo Exclusivo**).
>   - Si otro huésped concurrente intenta entrar, se topará con la puerta cerrada y tendrá que esperar de pie en el pasillo hasta que el primero salga (**Espera en Cola**).
>   - Si el huésped del pasillo tiene prisa, puede usar la política `NOWAIT`: toca la puerta una vez y, al ver que está ocupada, se va de inmediato a buscar un baño en el lobby en lugar de perder su tiempo esperando.
> 
> - **El Callejón sin Salida (Deadlock - Los Huéspedes Interbloqueados)**:
>   - Imagina al Huésped A y al Huésped B en una cena privada. Para comer, necesitan usar de forma obligatoria un **Cuchillo** y un **Tenedor**. Solo hay un juego disponible en la mesa.
>   - El **Huésped A** toma el **Cuchillo** y bloquea su uso.
>   - Al mismo microsegundo, el **Huésped B** toma el **Tenedor** y bloquea su uso.
>   - Ahora, el Huésped A se queda esperando pacientemente a que el Huésped B suelte el Tenedor para poder comer. Y el Huésped B se queda esperando a que el Huésped A suelte el Cuchillo.
>   - **La Parálisis**: Ninguno de los dos puede proceder, y ninguno está dispuesto a soltar el cubierto que ya tiene en su mano. Se quedarían esperando eternamente (**Deadlock**).
>   - En una base de datos relacional, el motor cuenta con un detector de deadlocks que analiza activamente este grafo de dependencias en bucle. Al detectar la parálisis, el motor "mata" inmediatamente a una de las dos transacciones (arrojando un error `40P01: deadlock_detected`), liberando sus recursos para que la otra transacción pueda finalizar con éxito.

---

## 5.3 Bloqueo Pesimista vs. Bloqueo Optimista

Cuando diseñamos concurrencia, nos enfrentamos a dos filosofías arquitectónicas opuestas para prevenir colisiones:

### 1. Bloqueo Pesimista (Pessimistic Locking)
* **Filosofía**: *"Asumo que los conflictos van a ocurrir con alta frecuencia. Por ende, bloqueo los registros de forma preventiva desde el inicio para que nadie más los toque mientras los proceso."*
* **Implementación**: Se realiza mediante base de datos utilizando `SELECT ... FOR UPDATE`.
* **Pros**: Consistencia absoluta garantizada en caliente; evita abortar transacciones largas al final.
* **Contras**: Reduce la concurrencia general del sistema y puede provocar parálisis (esperas de conexión) si las transacciones tardan mucho tiempo en procesar lógica del lado del servidor de aplicaciones.

### 2. Bloqueo Optimista (Optimistic Locking)
* **Filosofía**: *"Asumo que las colisiones concurrentes son raras. Dejo que todos lean y escriban libremente, pero al final de la operación valido si el registro cambió desde que lo leí. Si cambió, aborto la operación y pido al cliente que reintente."*
* **Implementación**: Se realiza a nivel de aplicación agregando una columna de versión (por ejemplo, un entero `version` o un `timestamp`).
* **Ejemplo SQL**:
  ```sql
  -- Paso 1: Leer el registro y su versión actual
  SELECT id, saldo, version FROM cuentas WHERE id = 10;
  -- saldo = 500, version = 4
  
  -- Paso 2: Ejecutar la actualización condicionando la versión previa
  UPDATE cuentas 
  SET saldo = 450, version = version + 1 
  WHERE id = 10 AND version = 4;
  ```
  Si otra transacción actualizó la cuenta antes de nuestro paso 2, el `UPDATE` afectará a `0` filas. La aplicación detecta esto, hace rollback y reintenta el flujo.
* **Pros**: Altamente escalable, no retiene conexiones ni bloqueos físicos en la base de datos.
* **Contras**: Requiere control explícito en el código de la aplicación y puede ser ineficiente (muchos reintentos abortados) ante escenarios de altísima contención de escrituras sobre las mismas filas.

---

## 5.4 Implementación en TypeScript de Bloqueos Seguros y Colas de Tareas (`SKIP LOCKED`)

A continuación, implementaremos un sistema de procesamiento de retiros de saldo financiero robusto que utiliza **Bloqueo Pesimista con protección contra esperas infinitas (`NOWAIT`)**, y un despachador de tareas asíncronas de alto rendimiento utilizando la potente cláusula **`SKIP LOCKED`**:

### `servicioBloqueos.ts`
```typescript
import { dbPool } from '../clients/dbClient';

// 1. Procesamiento transaccional de retiro utilizando Bloqueo Pesimista (FOR UPDATE NOWAIT)
export async function procesarRetiroPesimista(
  cuentaId: number,
  montoRetiro: number
): Promise<{ exito: boolean; mensaje: string }> {
  
  const cliente = await dbPool.connect();

  try {
    // Iniciar transacción
    await cliente.query('BEGIN');

    // Intentamos adquirir de inmediato un candado exclusivo sobre la cuenta.
    // Si otra transacción la tiene bloqueada, no esperaremos; fallaremos al instante.
    const sqlObtenerCuenta = `
      SELECT id, saldo 
      FROM cuentas 
      WHERE id = $1 
      FOR UPDATE NOWAIT
    `;
    
    const resultado = await cliente.query(sqlObtenerCuenta, [cuentaId]);

    if (resultado.rows.length === 0) {
      throw new Error('La cuenta no existe.');
    }

    const cuenta = resultado.rows[0];
    const saldoActual = parseFloat(cuenta.saldo);

    // Validación de negocio crítica
    if (saldoActual < montoRetiro) {
      throw new Error('Fondos insuficientes para completar el retiro.');
    }

    // Modificamos el registro con la total certeza de que nadie más puede interferir
    const sqlActualizarSaldo = `
      UPDATE cuentas 
      SET saldo = saldo - $1 
      WHERE id = $2
    `;
    await cliente.query(sqlActualizarSaldo, [montoRetiro, cuentaId]);

    // Consolidamos en disco liberando automáticamente el bloqueo físico
    await cliente.query('COMMIT');
    return { exito: true, mensaje: 'Retiro procesado exitosamente.' };

  } catch (error: any) {
    await cliente.query('ROLLBACK');

    // Código de error de PostgreSQL para bloqueo no disponible (debido a NOWAIT)
    if (error.code === '55P03') {
      return { 
        exito: false, 
        mensaje: 'La cuenta está siendo procesada en otra transacción activa. Por favor, reintente en unos segundos.' 
      };
    }

    return { exito: false, mensaje: error.message };
  } finally {
    cliente.release();
  }
}

// 2. Cola de tareas distribuidas de alto rendimiento usando SKIP LOCKED
export interface TareaProcesamiento {
  id: number;
  payload: string;
}

export async function consumirSiguienteTarea(): Promise<TareaProcesamiento | null> {
  const cliente = await dbPool.connect();

  try {
    await cliente.query('BEGIN');

    // Seleccionamos la siguiente tarea pendiente, bloqueándola en exclusiva FOR UPDATE.
    // Con SKIP LOCKED, si otros workers concurrentes ya bloquearon ciertas tareas,
    // las omitiremos limpiamente en lugar de quedarnos colgados en cola.
    const sqlConsumir = `
      SELECT id, payload 
      FROM cola_tareas 
      WHERE estado = 'pendiente' 
      ORDER BY creado_at ASC 
      LIMIT 1 
      FOR UPDATE SKIP LOCKED
    `;

    const resultado = await cliente.query(sqlConsumir);

    if (resultado.rows.length === 0) {
      // No hay tareas libres disponibles
      await cliente.query('COMMIT');
      return null;
    }

    const tarea = resultado.rows[0];

    // Marcamos la tarea como en proceso para que no sea seleccionada al liberar el commit
    const sqlActualizarEstado = `
      UPDATE cola_tareas 
      SET estado = 'en_proceso', procesado_at = NOW() 
      WHERE id = $1
    `;
    await cliente.query(sqlActualizarEstado, [tarea.id]);

    await cliente.query('COMMIT');
    return { id: tarea.id, payload: tarea.payload };

  } catch (error: any) {
    await cliente.query('ROLLBACK');
    console.error('Error procesando cola de tareas concurrentes:', error.message);
    return null;
  } finally {
    cliente.release();
  }
}
```

---

## Resumen del Capítulo

* Los **Bloqueos de Fila** (`FOR UPDATE` y `FOR SHARE`) previenen colisiones de escritura forzando a las transacciones concurrentes a encolarse secuencialmente.
* El modificador **`NOWAIT`** interrumpe inmediatamente una transacción con error si encuentra un registro bloqueado, protegiendo las conexiones de la aplicación de cuellos de botella por esperas infinitas.
* La cláusula **`SKIP LOCKED`** permite omitir de forma segura los registros bloqueados por otros hilos concurrentes, siendo el pilar por excelencia de colas de procesamiento asíncronas ultrarrápidas en base de datos.
* Un **Deadlock (Callejón sin Salida)** se origina cuando dos transacciones retienen recursos que la otra necesita para proceder. El motor relacional los detecta de forma activa resolviéndolos mediante el aborto automático de una de ellas.
* El **Bloqueo Optimista** basa su consistencia en el control de versiones en el lado de la aplicación, evitando retenciones físicas en la base de datos a costa de reintentos adicionales.

En el próximo capítulo, escalaremos nuestra destreza en SQL analítico y estructuración lógica mediante el estudio de **CTEs y Window Functions (Funciones de Ventana)** en profundidad.

---

[← Capítulo anterior (Capítulo 4)](04-niveles-aislamiento-y-mvcc.md) | [Inicio (README.md)](README.md) | [Capítulo siguiente (Capítulo 6) →](06-ctes-y-window-functions.md)
