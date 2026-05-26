# Capítulo 12: Procedimientos Almacenados, Triggers y PL/pgSQL

> "La base de datos relacional no es un archivador tonto de datos pasivos; es un entorno de computación altamente optimizado que puede ejecutar algoritmos transaccionales complejos a centímetros de distancia física del almacenamiento en disco, eliminando latencias de red destructivas."

En las arquitecturas tradicionales, los desarrolladores suelen adoptar el antipatrón de delegar absolutamente toda la lógica de negocio al servidor de aplicaciones (por ejemplo, a sus microservicios de Node.js o Go). Si para procesar una compra de stock necesitas: 1) validar stock, 2) restar stock, 3) registrar bitácora y 4) generar factura; realizar 4 viajes de red separados a la base de datos incrementa drásticamente la latencia y la posibilidad de fallos de red a la mitad de la transacción.

Para resolver esto de forma profesional, PostgreSQL nos ofrece **PL/pgSQL (Procedural Language/PostgreSQL)**, un lenguaje de programación estructurado integrado directamente en el motor relacional. En este capítulo, desmitificaremos la diferencia crucial entre **Funciones** y **Procedimientos Almacenados**, aprenderemos a dominar la programación reactiva mediante **Triggers** y escribiremos código PL/pgSQL de producción para auditorías automáticas de seguridad.

---

## 12.1 Programación en el Servidor: Funciones vs. Procedimientos

En PostgreSQL, contamos con dos abstracciones fundamentales para encapsular código procedimental en el servidor:

### 1. Funciones Almacenadas (`CREATE FUNCTION`)
* **Propósito**: Calcular y devolver un valor o un conjunto de registros de forma inmediata.
* **Restricción crítica**: **Las funciones corren de forma atómica dentro de la transacción del llamador y no pueden gestionar su propio ciclo de vida transaccional**. No se puede ejecutar un `COMMIT` o un `ROLLBACK` explícito dentro de una función clásica de PostgreSQL.
* **Uso típico**: Consultas complejas reutilizables, cálculos matemáticos o conversiones de datos.

### 2. Procedimientos Almacenados (`CREATE PROCEDURE`)
* **Propósito**: Ejecutar flujos de trabajo e instrucciones procedimentales complejas.
* **Súper poder**: **Los procedimientos sí pueden gestionar transacciones de forma autónoma**. Puedes abrir y cerrar transacciones físicamente en medio de la ejecución mediante sentencias `COMMIT` y `ROLLBACK` explícitas.
* **Uso típico**: Cargas masivas de datos por lotes (Batch processing) o flujos de compensación complejos.

---

## 12.2 Programación Reactiva en Disco: Triggers

Un **Trigger (Disparador)** es una regla automática configurada sobre una tabla que asocia la ejecución de una función PL/pgSQL ante eventos DML específicos: `INSERT`, `UPDATE` o `DELETE`.

### Momentos de Ejecución:
* **`BEFORE`**: El trigger se dispara antes de que los cambios se validen e intenten escribir en el disco. Es idóneo para validar datos (arrojando una excepción y cancelando la transacción) o para autocompletar columnas por defecto de forma dinámica.
* **`AFTER`**: El trigger se dispara tras consolidar la operación en disco. Excelente para propagar eventos colaterales, rellenar tablas históricas de auditoría o actualizar cachés de desnormalización.

### Variables Mágicas en PL/pgSQL:
Cuando se ejecuta un trigger, el motor relacional expone dos variables especiales de fila en el contexto local:
* **`NEW`**: Un registro que contiene los nuevos valores que se van a insertar o actualizar en la tabla.
* **`OLD`**: Un registro que contiene la versión original de la fila previa a sufrir un `UPDATE` o `DELETE`.

---

> [!NOTE]
> ### 🤖 El Robot Automatizado de la Cocina (Triggers)
> 
> Entendamos la programación en base de datos y los triggers con analogías físicas cotidianas:
> 
> - **El Enfoque Tradicional frente a PL/pgSQL (El Chef a Control Remoto)**:
>   - Imagina que eres un Chef Ejecutivo que vive en París (el **Servidor de Aplicaciones Node.js**), y tienes una cocina de restaurante ubicada en Tokio (el **Servidor de Base de Datos PostgreSQL**).
>   - Para preparar un platillo sencillo, le mandas un mensaje de texto al cocinero de Tokio: *"Paso 1: Saca 2 huevos del refrigerador"*. El cocinero lo hace y te responde por mensaje: *"Hecho"*. 
>   - Luego mandas otro mensaje: *"Paso 2: Rómpe los huevos en la sartén"*. Te responde: *"Hecho"*.
>   - Este intercambio continuo de mensajes por satélite en red tarda una eternidad debido a la distancia geográfica latente (**Latencia de Red**). Si el satélite pierde señal en el Paso 3, la sartén se quemará y la cocina colapsará.
>   - **El Enfoque PL/pgSQL**: Decides empaquetar toda la receta completa en una hoja de instrucciones física y se la dejas pegada en la nevera al cocinero en Tokio (**Función Almacenada**). 
>   - El cocinero ejecuta los 10 pasos de corrido a centímetros de los ingredientes sin mandarte un solo mensaje de texto hasta finalizar por completo el platillo. La receta se completa a la velocidad de la luz y de forma segura.
> 
> - **Los Triggers (El Sensor de Alarma de la Nevera)**:
>   - Decides instalar un robot automatizado de seguridad en la nevera de Tokio (**Trigger configurado en la tabla**).
>   - Configuras una regla reactiva: *"Cada vez que alguien abra la nevera para retirar un ingrediente (`BEFORE INSERT/UPDATE`), enciende la luz automáticamente e inspecciona la fecha de caducidad del producto"*.
>   - Si un empleado descuidado intenta meter carne en mal estado, el robot lo detecta a mitad de camino y le da un manotazo cerrando la puerta (**Arrojar una Excepción en el Trigger `BEFORE`**). La carne podrida nunca llega a tocar el estante de almacenamiento físico en disco.
>   - Si el ingrediente es correcto, se almacena. Al cerrarse la nevera, el robot apunta automáticamente en su bitácora digital quién retiró qué a qué hora (**Acción colateral en el Trigger `AFTER`**), garantizando que las auditorías ocurran de forma infalible sin que los humanos tengan que recordar apuntarlo.

---

## 12.3 Código PL/pgSQL Real: Auditoría Histórica Automática

A continuación, implementaremos un sistema de auditoría física completo utilizando **PL/pgSQL**. Crearemos una función disparadora que registra en una tabla histórica de auditorías todos los cambios microscópicos realizados en una tabla de saldos financieros, almacenando los valores anteriores (`OLD`) y nuevos (`NEW`) de forma automatizada:

### `auditoriaSaldos.sql`
```sql
-- 1. Crear la tabla de saldos financieros principal
CREATE TABLE cuentas_bancarias (
    id SERIAL PRIMARY KEY,
    usuario_id INT NOT NULL,
    saldo NUMERIC(15, 2) NOT NULL,
    actualizado_at TIMESTAMP DEFAULT NOW()
);

-- 2. Crear la tabla de auditoría histórica
CREATE TABLE auditoria_cuentas (
    id SERIAL PRIMARY KEY,
    cuenta_id INT NOT NULL,
    saldo_anterior NUMERIC(15, 2),
    saldo_nuevo NUMERIC(15, 2),
    operacion VARCHAR(10) NOT NULL, -- 'INSERT', 'UPDATE', 'DELETE'
    usuario_db VARCHAR(50) DEFAULT CURRENT_USER,
    realizado_at TIMESTAMP DEFAULT NOW()
);

-- 3. Crear la función disparadora PL/pgSQL
CREATE OR REPLACE FUNCTION registrar_cambio_saldo()
RETURNS TRIGGER AS $$
BEGIN
    -- Evaluamos la acción del evento DML
    IF (TG_OP = 'UPDATE') THEN
        -- Validamos que el saldo haya cambiado realmente para evitar ruido en la bitácora
        IF OLD.saldo IS DISTINCT FROM NEW.saldo THEN
            INSERT INTO auditoria_cuentas (cuenta_id, saldo_anterior, saldo_nuevo, operacion)
            VALUES (OLD.id, OLD.saldo, NEW.saldo, 'UPDATE');
        END IF;
        
        -- Actualizamos automáticamente la fecha de modificación en el registro principal
        NEW.actualizado_at = NOW();
        RETURN NEW;

    ELSIF (TG_OP = 'INSERT') THEN
        INSERT INTO auditoria_cuentas (cuenta_id, saldo_anterior, saldo_nuevo, operacion)
        VALUES (NEW.id, NULL, NEW.saldo, 'INSERT');
        RETURN NEW;

    ELSIF (TG_OP = 'DELETE') THEN
        INSERT INTO auditoria_cuentas (cuenta_id, saldo_anterior, saldo_nuevo, operacion)
        VALUES (OLD.id, OLD.saldo, NULL, 'DELETE');
        RETURN OLD;
    END IF;
    
    RETURN NULL; -- Indica que el trigger completó con éxito
END;
$$ LANGUAGE plpgsql;

-- 4. Asociar la función disparadora mediante un TRIGGER
-- Configuramos el trigger para ejecutarse por cada fila modificada (FOR EACH ROW)
CREATE TRIGGER trg_auditoria_cuentas
AFTER INSERT OR UPDATE OR DELETE
ON cuentas_bancarias
FOR EACH ROW
EXECUTE FUNCTION registrar_cambio_saldo();
```

### `servicioAuditoriaWrapper.ts`
```typescript
import { dbPool } from '../clients/dbClient';

export interface RegistroAuditoria {
  id: number;
  cuentaId: number;
  saldoAnterior: number | null;
  saldoNuevo: number | null;
  operacion: string;
  usuarioDb: string;
  realizadoAt: Date;
}

// 1. wrapper TypeScript para simular una actualización ordinaria de saldo
// Demuestra cómo el trigger físico de PostgreSQL audita y guarda logs en background sin que el backend haga nada extra
export async function actualizarSaldoTransaccional(
  cuentaId: number,
  nuevoSaldo: number
): Promise<boolean> {
  const cliente = await dbPool.connect();

  try {
    await cliente.query('BEGIN');

    // Ejecutamos un simple UPDATE ordinario
    const sqlActualizar = `
      UPDATE cuentas_bancarias 
      SET saldo = $1 
      WHERE id = $2
    `;
    const resultado = await cliente.query(sqlActualizar, [nuevoSaldo, cuentaId]);

    if (resultado.rowCount === 0) {
      throw new Error(`No se encontró la cuenta con ID ${cuentaId}`);
    }

    await cliente.query('COMMIT');
    console.log(`[Bitácora] Saldo de la cuenta ${cuentaId} actualizado a ${nuevoSaldo}. El trigger auditó de forma invisible en la base de datos.`);
    return true;

  } catch (error: any) {
    await cliente.query('ROLLBACK');
    console.error('Error al actualizar el saldo financiero:', error.message);
    return false;
  } finally {
    cliente.release();
  }
}

// 2. Consultar el historial de auditoría generado reactivamente por el trigger
export async function obtenerLogsAuditoria(cuentaId: number): Promise<RegistroAuditoria[]> {
  const cliente = await dbPool.connect();

  try {
    const sqlLogs = `
      SELECT id, cuenta_id AS "cuentaId", saldo_anterior AS "saldoAnterior", 
             saldo_nuevo AS "saldoNuevo", operacion, usuario_db AS "usuarioDb", realizado_at AS "realizadoAt"
      FROM auditoria_cuentas
      WHERE cuenta_id = $1
      ORDER BY realizado_at DESC
    `;
    const resultado = await cliente.query(sqlLogs, [cuentaId]);
    
    return resultado.rows.map(row => ({
      id: row.id,
      cuentaId: row.cuentaId,
      saldoAnterior: row.saldoAnterior ? parseFloat(row.saldoAnterior) : null,
      saldoNuevo: row.saldoNuevo ? parseFloat(row.saldoNuevo) : null,
      operacion: row.operacion,
      usuarioDb: row.usuarioDb,
      realizadoAt: row.realizadoAt
    }));

  } catch (error: any) {
    console.error('Error al consultar historial de auditoría física:', error.message);
    throw error;
  } finally {
    cliente.release();
  }
}
```

---

## Resumen del Capítulo

* **PL/pgSQL** es el lenguaje estructurado de PostgreSQL que permite ejecutar computación procedimental compleja directamente en el clúster de base de datos a centímetros del disco.
* Las **Funciones Almacenadas** se ejecutan de forma atómica dentro de la transacción activa del llamador. Los **Procedimientos Almacenados** rompen este límite y permiten abrir y cerrar transacciones en caliente mediante `COMMIT` y `ROLLBACK`.
* Los **Triggers** actúan como centinelas automáticos que reaccionan a eventos DML (`INSERT`, `UPDATE`, `DELETE`). Los triggers `BEFORE` son ideales para validaciones estrictas y mutación de datos; los triggers `AFTER` destacan al propagar bitácoras de auditoría e integraciones colaterales.
* La programación procedural alivia cuellos de botella masivos de red al unificar procesos secuenciales de múltiples pasos en un solo viaje de ida y vuelta a la base de datos.

En el próximo capítulo, abordaremos uno de los retos de ingeniería de producción más temidos de la industria: cómo desplegar e implementar **Migraciones de Esquema Zero-Downtime** sin detener tu servicio en caliente.

---

[← Capítulo anterior (Capítulo 11)](11-seguridad-y-rls.md) | [Inicio (README.md)](README.md) | [Capítulo siguiente (Capítulo 13) →](13-migraciones-zero-downtime.md)
