# Capítulo 4: Niveles de Aislamiento y Anomalías de Concurrencia

> "En sistemas distribuidos y concurrentes, la consistencia absoluta no se pierde por errores de sintaxis; se destruye cuando dos transacciones correctas se ejecutan al mismo tiempo bajo un nivel de aislamiento defectuoso."

Cuando diseñas un sistema relacional listo para producción, debes asumir que tu base de datos recibirá cientos de peticiones de lectura y escritura concurrentes por segundo. El motor relacional debe procesar estas transacciones de forma paralela en hilos independientes compartiendo los mismos bloques de disco. 

Para evitar el caos y la corrupción de datos, el estándar SQL define los **Niveles de Aislamiento transaccional**. Sin embargo, la gran mayoría de los programadores desconocen el funcionamiento microscópico del motor físico, configurando niveles predeterminados que dan pie a anomalías catastróficas en caliente. En este capítulo, desmitificaremos las anomalías de lectura, el peligroso **Write Skew**, el funcionamiento de **MVCC (Multi-Version Concurrency Control)** en PostgreSQL y las tripas del proceso **VACUUM**.

---

## 4.1 Fenómenos de Lectura y Aislamiento Estándar

El estándar SQL define cuatro niveles de aislamiento transaccional que protegen al sistema contra tres fenómenos clásicos de lectura concurrentes:

### 1. Los Fenómenos de Concurrencia
* **Lectura Sucia (Dirty Read)**: Ocurre cuando la Transacción A lee modificaciones intermedias realizadas por la Transacción B antes de que esta última haga `COMMIT`. Si B hace un `ROLLBACK`, la Transacción A habrá tomado decisiones basadas en datos fantasmas e inexistentes.
* **Lectura No Repetible (Non-Repeatable Read)**: Ocurre cuando la Transacción A lee una fila, la Transacción B modifica esa misma fila y hace `COMMIT`, y la Transacción A vuelve a leer la misma fila obteniendo valores discrepantes en medio de su misma transacción.
* **Lectura Fantasma (Phantom Read)**: Ocurre cuando la Transacción A ejecuta una búsqueda de rango (por ejemplo, *usuarios con edad > 30*), la Transacción B inserta una nueva fila que cumple el criterio y hace `COMMIT`, y la Transacción A vuelve a consultar el rango obteniendo filas nuevas ("fantasmas") que antes no existían.

### 2. Matriz de Niveles de Aislamiento Estándar:

| Nivel de Aislamiento | Lectura Sucia | Lectura No Repetible | Lectura Fantasma |
| :--- | :--- | :--- | :--- |
| **Read Uncommitted** | Permitida | Permitida | Permitida |
| **Read Committed** (Predeterminado) | Protegida | Permitida | Permitida |
| **Repeatable Read** | Protegida | Protegida | Permitida (PostgreSQL la protege) |
| **Serializable** (Consistencia Total) | Protegida | Protegida | Protegida |

---

## 4.2 La Anomalía Definitiva: Write Skew (Sesgo de Escritura)

Incluso en el nivel **Repeatable Read** (que en PostgreSQL bloquea de forma nativa las lecturas fantasma), el sistema sigue siendo vulnerable a la anomalía concurrente más sutil y destructiva del software relacional: el **Write Skew (Sesgo de Escritura)**.

Ocurre cuando dos transacciones concurrentes leen un conjunto de datos común que cumple con una restricción global, toman decisiones complementarias e independientes basadas en esa lectura, y escriben en registros separados. Al hacer `COMMIT`, ambas transacciones tienen éxito localmente, pero juntas **violan la restricción de integridad global del negocio**.

> [!NOTE]
> ### 🎥 La Cámara del Tiempo de Múltiples Dimensiones y el Conflicto de los Médicos
> 
> Visualicemos el MVCC y la anomalía Write Skew con analogías didácticas claras:
> 
> - **El Funcionamiento de MVCC (Cámara de Tiempo de Múltiples Dimensiones)**:
>   - Imagina que estás leyendo un libro de historia de 1,000 páginas en la biblioteca. De pronto, un corrector de estilo (la Transacción de Escritura) llega a modificar la página 45.
>   - En una base de datos antigua con bloqueos rígidos, el corrector te quitaría el libro de las manos y te obligaría a esperar de pie hasta que termine de borrar y escribir (**Bloqueos de Lectura**).
>   - En **PostgreSQL (MVCC)**, el corrector nunca altera tu libro. Toma la página 45, le saca una **fotocopia física instantánea**, realiza las modificaciones en su copia y le añade una marca invisible de tiempo: *"Visible a partir del minuto 10:00 (su `xmin`)"*.
>   - Tú (la Transacción de Lectura) sigues leyendo la página original de tu libro en tu propia dimensión temporal sin interrupciones ni bloqueos de red. Las lecturas nunca bloquean a las escrituras, y las escrituras nunca bloquean a las lecturas.
> - **El Write Skew (La Colisión de los Médicos de Guardia)**:
>   - Una clínica médica tiene una regla de oro de supervivencia: *"Al menos un médico activo debe quedarse de guardia física en la clínica en todo momento"*.
>   - Hoy están de guardia el **Médico A** y el **Médico B**. Ambos se sienten cansados y quieren irse a casa a dormir.
>   - Concurrentemente, el Médico A abre la app en su móvil (Transacción 1) y consulta: *¿Cuántos médicos activos hay de guardia?* La pantalla lee del disco: *2 médicos activos (A y B)*. Como hay otro médico (B), la regla de oro se cumple. El Médico A solicita permiso para irse a dormir (Modifica su estado a "inactivo").
>   - Al mismo microsegundo exacto, el Médico B abre su app (Transacción 2) y consulta: *¿Cuántos médicos activos hay de guardia?* En su dimensión temporal Repeatable Read, el Médico A sigue activo. La app lee: *2 médicos activos*. Como el Médico A está activo, el Médico B solicita irse a dormir (Modifica su estado a "inactivo").
>   - Ambas transacciones hacen `COMMIT` con éxito porque en sus respectivas lecturas iniciales la regla de oro no se rompía.
>   - **El Desastre**: Al consolidarse ambas escrituras en disco, la clínica se queda con **0 médicos de guardia**. La regla de oro global se ha roto por completo sin que el motor arrojara ningún error de concurrencia clásica. Solo el nivel de aislamiento **Serializable** detectaría que las historias causales chocan, abortando una de las transacciones.

---

## 4.3 Internals de MVCC en PostgreSQL: xmin y xmax

Para lograr que las lecturas no bloqueen escrituras y viceversa, PostgreSQL no sobrescribe físicamente tus registros en disco durante un `UPDATE`. Implementa **MVCC (Control de Concurrencia Multiversión)**.

Cada fila (tupla física) en el disco duro contiene metadatos ocultos en su cabecera de fila de 23 bytes llamada **`HeapTupleHeaderData`**:
* **`xmin`**: El ID de la transacción física que insertó originalmente la tupla.
* **`xmax`**: El ID de la transacción física que eliminó o actualizó (marcando como obsoleta) la tupla. Si la tupla sigue activa y no se ha modificado, `xmax` es `0`.
* **`t_infomask` y `t_infomask2`**: Flags de bits binarios cruciales en la cabecera. PostgreSQL los utiliza para almacenar el estado de la transacción (si se hizo `COMMIT` o `ROLLBACK`) directamente en la fila. **Esto evita que el motor tenga que consultar en red las tablas de estado del sistema (como el commit log CLOG) en cada lectura**, acelerando las comprobaciones de visibilidad en microsegundos.

### La Anatomía de un Visibility Snapshot
Cuando ejecutas una transacción, PostgreSQL genera un **Snapshot de Visibilidad** en memoria para definir qué datos son visibles para ti. El formato de este snapshot es:

$$\text{xmin} : \text{xmax} : \text{active\_list}$$

Por ejemplo: `100:120:105,108` representa:
* **`xmin` ($100$)**: Todas las transacciones inferiores a $100$ están consolidadas (`committed`) y sus cambios son visibles.
* **`xmax` ($120$)**: Todas las transacciones superiores o iguales a $120$ no han arrancado o siguen activas y sus cambios son completamente invisibles para ti.
* **`active_list` ($105, 108$)**: Lista de transacciones intermedias que están activas actualmente. Aunque sus IDs sean menores que $120$, sus cambios no son visibles porque no han hecho `COMMIT` en tu línea temporal.

### El flujo físico de un `UPDATE` en disco y HOT Updates
Cuando ejecutas `UPDATE usuarios SET saldo = 100 WHERE id = 1;` bajo la transacción ID `500`:
1. PostgreSQL **no borra** la fila antigua del disco. Simplemente escribe en el campo `xmax` de la fila antigua el ID de tu transacción: `xmax = 500`. Esto la marca como "muerta" o invisible para cualquier transacción nueva.
2. Escribe una **tupla física completamente nueva** en otra sección del disco con el nuevo valor del saldo, definiendo en su cabecera `xmin = 500` y `xmax = 0`.
3. Durante unos instantes, coexisten dos versiones físicas de la misma fila en el disco duro.

#### El Secreto de Optimización: HOT (Heap-Only Tuple) Updates
Normalmente, duplicar una tupla física obliga al motor a insertar un puntero nuevo en todos los índices B-Tree asociados a la tabla. Esto causa un overhead severo de I/O de disco.
* **Mecánica HOT**: Si realizas un `UPDATE` sobre columnas que **no están indexadas** y hay espacio libre suficiente dentro de la **misma página física de 8KB** de disco:
  1. PostgreSQL escribe la tupla nueva dentro de la misma página de 8KB.
  2. Crea un puntero de enlace directo desde la tupla antigua hacia la nueva.
  3. **No toca ningún índice B-Tree**. Los índices siguen apuntando a la tupla antigua, y el motor sigue el enlace directo de forma automática en memoria RAM.
  4. Esto reduce la degradación de escrituras a cero y permite una autolimpieza instantánea.

### El rol vital de VACUUM
Como PostgreSQL acumula tuplas "muertas" (registros obsoletos que sufrieron `UPDATE` o `DELETE`) continuamente en disco, los archivos físicos crecerían sin límite degradando el rendimiento. 

Aquí entra el proceso **VACUUM**:
* Analiza de forma asíncrona las páginas del disco duro buscando tuplas muertas que ya no sean visibles para ninguna transacción activa del sistema.
* Borra físicamente las referencias de esas tuplas y marca esos sectores del archivo en disco como "libres", permitiendo que futuras inserciones reutilicen ese espacio de bytes sin tener que expandir el tamaño del archivo físico en el sistema de archivos del sistema operativo.
* Si el proceso de autovacuum de PostgreSQL no está bien configurado o tus transacciones duran horas abiertas, las tuplas muertas se acumularán disparando el tamaño físico en disco de forma alarmante (**Table Bloat**).

---

## 4.4 Implementación en TypeScript de Aislamiento Serializable

Implementemos un servicio en TypeScript que ejecute transferencias bancarias críticas utilizando el nivel de aislamiento máximo **Serializable**, incorporando de forma obligatoria un bucle de reintentos asíncrono para gestionar los abortos lógicos que PostgreSQL disparará si detecta colisiones de concurrencia o Write Skew:

### `transaccionSerializable.ts`
```typescript
import { dbPool } from '../clients/dbClient';

// Procesar transacción crítica bajo aislamiento Serializable con reintentos automáticos
export async function procesarOperacionSerializable(
  emisorId: number,
  monto: number,
  reintentosMaximos: number = 3
): Promise<boolean> {
  
  let intentos = 0;

  while (intentos < reintentosMaximos) {
    const cliente = await dbPool.connect();
    
    try {
      // 1. Iniciar la transacción configurando el nivel de aislamiento Serializable
      await cliente.query('BEGIN TRANSACTION ISOLATION LEVEL SERIALIZABLE');

      // 2. Realizar verificaciones lógicas (ej: saldo acumulado de retiros diarios)
      const queryRetiros = `
        SELECT COALESCE(SUM(monto), 0) AS total_diario 
        FROM transacciones_diarias 
        WHERE usuario_id = $1 AND fecha = CURRENT_DATE
      `;
      const resultado = await cliente.query(queryRetiros, [emisorId]);
      const totalDiario = parseFloat(resultado.rows[0].total_diario);

      // Restricción global: no retirar más de $5,000 USD al día
      if (totalDiario + monto > 5000) {
        throw new Error('Límite de retiro diario superado.');
      }

      // 3. Registrar el nuevo retiro si cumple
      await cliente.query(
        'INSERT INTO transacciones_diarias (usuario_id, monto, fecha) VALUES ($1, $2, CURRENT_DATE)',
        [emisorId, monto]
      );

      // 4. Intentar consolidar en disco
      await cliente.query('COMMIT');
      return true; // Transacción completada con éxito absoluto

    } catch (error: any) {
      // 5. Rollback preventivo e inmediato ante fallos
      await cliente.query('ROLLBACK');

      // Código de error SQL 40001: Representa un fallo de serialización por concurrencia activa
      if (error.code === '40001') {
        intentos++;
        console.warn(`[Serializable-Retry] Colisión concurrente detectada. Reintentando intento ${intentos}/${reintentosMaximos}...`);
        
        // Esperamos un backoff exponencial de milisegundos aleatorio para evitar colisiones repetidas
        await new Promise(resolve => setTimeout(resolve, Math.random() * 100 * intentos));
      } else {
        // Si es un error de lógica de negocio (límite superado), no reintentamos y abortamos
        console.error('[Serializable-Abort] Error de lógica o esquema:', error.message);
        return false;
      }
    } finally {
      cliente.release();
    }
  }

  console.error('[Serializable-Failure] Transacción fallida tras agotar reintentos por colisiones concurrentes.');
  return false;
}
```

---

## Resumen del Capítulo

* Las anomalías de lectura clásicas son **Lectura Sucia**, **Lectura No Repetible** y **Lectura Fantasma**, controladas mediante la escala de aislamientos del estándar SQL.
* El **Write Skew (Sesgo de Escritura)** es la anomalía definitiva: transacciones complementarias independientes violan juntas una restricción de integridad global al escribir en registros disjuntos. Solo se erradica bajo el aislamiento **Serializable** con bucles de reintentos en el cliente.
* PostgreSQL implementa **MVCC** duplicando en disco nuevas versiones físicas de tuplas enriquecidas con metadatos de visibilidad temporal **`xmin`** y **`xmax`**, evitando que las lecturas bloqueen escrituras.
* El proceso **VACUUM** limpia y reclama asíncronamente el espacio en disco de las tuplas muertas obsoletas, previniendo el Table Bloat en producción.

En el próximo capítulo, estudiaremos el motor físico de la contención: los **Mecanismos de Bloqueos (Locking)**, desmitificando el uso de `FOR UPDATE`, bloqueos de tabla y prevención de Deadlocks.

---

[← Capítulo anterior (Capítulo 3)](03-transacciones-y-acid.md) | [Inicio (README.md)](README.md) | [Capítulo siguiente (Capítulo 5) →](05-bloqueos-locking.md)
