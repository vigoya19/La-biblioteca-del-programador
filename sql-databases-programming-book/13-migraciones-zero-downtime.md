# Capítulo 13: Migraciones de Esquema Zero-Downtime

> "Modificar la estructura lógica de una base de datos con millones de filas y miles de consultas por segundo activas es una de las maniobras más peligrosas en ingeniería. Un simple ALTER TABLE descuidado bloqueará absolutamente todo el sistema, tirando abajo el servicio en caliente. La excelencia técnica se mide por la capacidad de desplegar cambios estructurales con cero segundos de inactividad."

En entornos de startups y corporaciones modernas, la velocidad de despliegue es vital: lanzamos nuevas características, refactorizamos modelos y modificamos esquemas de base de datos de forma continua. Sin embargo, a medida que las tablas crecen, las migraciones ordinarias de SQL se convierten en bombas de tiempo listas para colapsar producción.

En este capítulo, aprenderemos las reglas de oro físicas para realizar **Migraciones Zero-Downtime** en PostgreSQL. Estudiaremos cómo agregar columnas con valores por defecto de forma segura, el peligro invisible de los bloqueos exclusivos largos y los **Lock Timeouts**, cómo construir índices en caliente usando **`CONCURRENTLY`**, y diseñaremos un plan de migración paso a paso en TypeScript para producción sin interrumpir el servicio.

---

## 13.1 El Peligro Invisible: ACCESS EXCLUSIVE Locks

Como estudiamos en el Capítulo 5, comandos DDL como `ALTER TABLE` adquieren de forma inmediata un candado físico destructivo llamado **`ACCESS EXCLUSIVE LOCK`** sobre la tabla afectada.
* Este candado **bloquea absolutamente todo**: impide que cualquier otra transacción concurrente ejecute un `SELECT`, `INSERT`, `UPDATE` o `DELETE` sobre la tabla.
* Si la tabla tiene 50 millones de filas y ejecutas un `ALTER TABLE` que obliga a reescribir físicamente cada fila en el disco (como cambiar un tipo de dato de `INT` a `BIGINT`), el bloqueo exclusivo se retendrá por varios minutos, congelando todas las conexiones de tu backend y tirando el sitio por completo.

### La regla de oro del Lock Timeout:
Por defecto, si una migración no puede adquirir el bloqueo exclusivo porque otra consulta de lectura larga está activa, la migración esperará eternamente en cola. **Esto encola de forma inmediata a todas las lecturas posteriores**, provocando la parálisis total de tu pool de conexiones.

Para solucionarlo defensivamente, **siempre debemos configurar un Lock Timeout muy restrictivo** antes de cualquier sentencia DDL. Si la migración no puede bloquear la tabla en 2 segundos, aborta de inmediato liberando la cola y permitiendo que el tráfico fluya, para volver a intentarlo en un momento de menor actividad:
```sql
-- Configurar el tiempo máximo de espera de bloqueo exclusivo a 2 segundos
SET lock_timeout = '2s';
ALTER TABLE usuarios ADD COLUMN telefono VARCHAR(20);
```

---

## 13.2 Patrones de Migración Zero-Downtime Clave

Para evolucionar un esquema de producción en caliente sin provocar caídas, debemos dividir los cambios destructivos grandes en múltiples fases atómicas inofensivas:

### 1. Agregar una Columna con un Valor por Defecto (`DEFAULT`)
* **Antipatrón (Peligroso)**: 
  ```sql
  -- En PostgreSQL 10 o anterior, esto bloqueaba y reescribía físicamente toda la tabla en disco para inyectar el valor por defecto en cada tupla
  ALTER TABLE pedidos ADD COLUMN estado VARCHAR(20) DEFAULT 'pendiente' NOT NULL;
  ```
* **Patrón Moderno (PostgreSQL 11+)**: PostgreSQL 11 resolvió esto de forma nativa. Agregar una columna con un valor por defecto no reescribe la tabla; almacena el valor por defecto en los metadatos de la tabla de forma instantánea.
* **Si usas restricciones estrictas o PostgreSQL antiguo, el flujo de tres pasos es obligatorio**:
  1. Agregar la columna permitiendo nulos (`NULL`) de forma instantánea.
  2. Rellenar los valores por defecto en lotes pequeños (Batches) utilizando un script de background en TypeScript para no saturar los logs de disco (`WAL`).
  3. Aplicar la restricción `NOT NULL` con validaciones progresivas.

### 2. Crear Índices de Forma Segura: `CONCURRENTLY`
* **Antipatrón (Peligroso)**:
  ```sql
  -- Bloquea la tabla para escrituras (SHARE LOCK) hasta que termine de construirse el índice completo, lo cual puede tardar horas.
  CREATE INDEX idx_pedidos_usuario ON pedidos(usuario_id);
  ```
* **Patrón Zero-Downtime**:
  ```sql
  -- Construye el índice recorriendo la tabla dos veces de fondo de forma asíncrona,
  -- permitiendo que todas las lecturas y escrituras concurrentes continúen fluyendo.
  CREATE INDEX CONCURRENTLY idx_pedidos_usuario ON pedidos(usuario_id);
  ```
  > [!IMPORTANT]
  > Las creaciones de índices concurrentes no pueden ejecutarse dentro de bloques transaccionales (`BEGIN...COMMIT`). El driver del cliente debe disparar la consulta fuera de una transacción explícita.

---

> [!NOTE]
> ### 🏎️ Cambiar las Llantas del Coche de F1 en Plena Carrera
> 
> Entendamos las migraciones zero-downtime, los bloqueos exclusivos y la creación concurrente de índices con una analogía física deportiva:
> 
> - **El Antipatrón de Migración Rígida (Detener el Coche en Mitad de la Autopista)**:
>   - Imagina que tu coche de Fórmula 1 está compitiendo a 300 km/h en un circuito de carreras (la base de datos procesando transacciones concurrentes).
>   - De pronto, el ingeniero de pista te avisa que las llantas están desgastadas y hay que cambiarlas (**Migración de Base de Datos**).
>   - Si aplicas el antipatrón de bloqueo exclusivo salvaje, el mecánico decide saltar al asfalto en mitad de la pista y cruzarse enfrente de tu coche obligándote a clavar los frenos de golpe a 300 km/h (**ACCESS EXCLUSIVE LOCK**).
>   - Detienes el coche a media pista, provocando que los coches de atrás choquen y se arme un tapón vial de kilómetros en caliente (parálisis de conexiones). Tardas 3 minutos en cambiar las llantas y luego vuelves a arrancar. El negocio perdió la carrera y el sitio web se cayó.
> 
> - **El Enfoque Zero-Downtime (La Parada en Boxes Coordinada - Pit Stop)**:
>   - En lugar de detenerte a mitad de la pista, el ingeniero diseña una estrategia atómica de fases fluidas.
>   - Primero, construyes una vía alterna pavimentada paralela a la pista principal (**Crear Columna Temporal de Soporte**).
>   - Los coches siguen circulando a toda velocidad por la pista principal sin enterarse de nada.
>   - Mientras tanto, el equipo de mecánicos prepara las llantas y los taladros neumáticos en los boxes de forma aislada (**Construcción de Índice Concurrente `CONCURRENTLY`**).
>   - Cuando todo está listo de forma asíncrona, el coche entra de forma fluida a la bahía de boxes, se cambian las 4 llantas en 2.1 segundos exactos y sale al asfalto a toda velocidad sin interrumpir la circulación de ningún otro coche del circuito. El cambio estructural se consolidó con éxito sin detener la carrera del negocio ni un solo segundo.

---

## 13.3 Implementación de un Script TypeScript de Migración Concurrente Segura

A continuación, implementaremos un script orquestador en TypeScript diseñado para desplegar una migración en producción. El script configura de forma defensiva un **Lock Timeout** antes de realizar alteraciones de tabla, gestiona el reintento automático exponencial ante bloqueos activos, y ejecuta una creación de índice concurrente fuera del bloque transaccional para garantizar que la base de datos nunca sufra caídas:

### `orquestadorMigracion.ts`
```typescript
import { dbPool } from '../clients/dbClient';

// Configurar reintentos asíncronos para migraciones DDL con protección de Lock Timeout
export async function ejecutarMigracionDDLSegura(
  sqlAlteracion: string,
  reintentosMaximos: number = 5,
  esperaBaseMs: number = 2000
): Promise<boolean> {
  
  let intentos = 0;

  while (intentos < reintentosMaximos) {
    const cliente = await dbPool.connect();

    try {
      await cliente.query('BEGIN');

      // 1. Configurar de forma estricta un Lock Timeout de 2 segundos.
      // Si la base de datos está bajo alta carga y no puede adquirir el ACCESS EXCLUSIVE lock
      // en este lapso, abortará inmediatamente en lugar de colgar el sistema entero.
      await cliente.query("SET LOCAL lock_timeout = '2000'");

      console.log(`[Migraciones-DDL] Intentando aplicar alteración estructural (Intento ${intentos + 1}/${reintentosMaximos})...`);
      
      // 2. Ejecutar la sentencia de alteración
      await cliente.query(sqlAlteracion);

      await cliente.query('COMMIT');
      console.log('[Migraciones-DDL] Alteración de esquema aplicada de forma exitosa y segura.');
      return true;

    } catch (error: any) {
      await cliente.query('ROLLBACK');

      // Código SQL 55P03 indica que se excedió el lock timeout configurado
      if (error.code === '55P03') {
        intentos++;
        const espera = esperaBaseMs * Math.pow(2, intentos); // Backoff exponencial
        console.warn(`[Migraciones-DDL] No se pudo adquirir el bloqueo exclusivo en 2 segundos por contención concurrente. Reintentando en ${espera}ms...`);
        await new Promise(resolve => setTimeout(resolve, espera));
      } else {
        // Si es otro error de sintaxis o lógica, abortamos inmediatamente
        console.error('[Migraciones-DDL] Fallo catastrófico no recuperable en migración:', error.message);
        return false;
      }
    } finally {
      cliente.release();
    }
  }

  console.error('[Migraciones-DDL] Migración abortada permanentemente tras agotar los reintentos para proteger la producción.');
  return false;
}

// 3. Crear índice de forma concurrentemente fuera de transacciones
export async function crearIndiceConcurrente(
  nombreIndice: string,
  tabla: string,
  columna: string
): Promise<boolean> {
  // Las creaciones concurrentes DEBEN ejecutarse en un cliente dedicado del pool sin usar BEGIN/COMMIT
  const cliente = await dbPool.connect();

  try {
    console.log(`[Migraciones-Index] Iniciando construcción asíncrona de índice '${nombreIndice}' en caliente...`);
    
    // Ejecutamos CREATE INDEX CONCURRENTLY de forma aislada
    const sqlCrear = `CREATE INDEX CONCURRENTLY IF NOT EXISTS ${nombreIndice} ON ${tabla}(${columna})`;
    await cliente.query(sqlCrear);
    
    console.log(`[Migraciones-Index] Índice '${nombreIndice}' construido con éxito sin bloqueos de lectura/escritura.`);
    return true;

  } catch (error: any) {
    console.error(`[Migraciones-Index] Error al crear índice de forma concurrente:`, error.message);
    return false;
  } finally {
    cliente.release();
  }
}
```

---

## Resumen del Capítulo

* Las alteraciones estructurales adquieren un bloqueo exclusivo **`ACCESS EXCLUSIVE LOCK`** que colapsa la concurrencia de la tabla entera si no se maneja de forma adecuada.
* Configurar defensivamente un **`lock_timeout`** restrictivo evita la acumulación de transacciones en cola cuando una migración se cuelga esperando bloqueos en horas pico.
* La cláusula **`CONCURRENTLY`** es obligatoria al crear índices en producción, permitiendo que PostgreSQL los compile en segundo plano sin interrumpir las lecturas ni escrituras concurrentes.
* El diseño de migraciones Zero-Downtime exige romper cambios estructurales destructivos en pequeñas etapas lógicas y transiciones progresivas en caliente.

En el próximo capítulo, consolidaremos e integraremos todo lo aprendido a lo largo de este volumen mediante el desarrollo de nuestro **Proyecto Integrador: Backend SaaS Multi-Tenant** de nivel empresarial.

---

[← Capítulo anterior (Capítulo 12)](12-stored-procedures-y-triggers.md) | [Inicio (README.md)](README.md) | [Capítulo siguiente (Capítulo 14) →](14-proyecto-practico-saas.md)
