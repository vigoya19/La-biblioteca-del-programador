# Capítulo 14: Proyecto Práctico Integrador: SaaS Multi-Tenant

> "La culminación del diseño relacional no es la memorización de comandos de consola. Consiste en la arquitectura armónica de código TypeScript robusto y políticas de seguridad a nivel de base de datos para crear un sistema SaaS multi-tenant unificado que soporte millones de transacciones con total aislamiento lógico y la máxima velocidad física."

A lo largo de este libro, hemos deconstruido los pilares microscópicos de las bases de datos relacionales: desde el planificador físico de consultas y el diseño del almacenamiento hasta las garantías transaccionales, control de concurrencia mediante bloqueos, replicaciones robustas, RLS y migraciones sin inactividad. 

En este capítulo integrador, consolidaremos todo este conocimiento construyendo una **Arquitectura de Software SaaS Multi-Tenant de Grado Producción**. Implementaremos un backend completo en TypeScript que automatiza de forma atómica la creación de inquilinos (`tenants`), gestiona transacciones concurrentes complejas, establece variables locales de sesión transaccionales y aplica **Row-Level Security (RLS)** de forma impenetrable sobre el clúster físico.

---

## 14.1 Arquitectura del Rascacielos Compartido

En el desarrollo de software tipo Software as a Service (SaaS), nos enfrentamos a tres enfoques para organizar la base de datos de los inquilinos:
1. **Base de Datos Separada por Tenant**: Máximo aislamiento físico, pero extremadamente costoso de mantener y difícil de escalar cuando tienes miles de pequeños tenants gratuitos.
2. **Esquema Separado por Tenant**: Tablas idénticas en esquemas de PostgreSQL separados (`CREATE SCHEMA tenant_a`). Un término medio aceptable pero complejo de migrar en masa.
3. **Base de Datos Compartida con Aislamiento RLS (Single-Database Multi-Tenant)**: Todos los datos conviven en el mismo disco, en las mismas tablas físicas compartidas, diferenciados por una columna `tenant_id`. Es el enfoque **más rentable, mantenible y escalable**, y gracias a **Row-Level Security (RLS)**, garantizamos un aislamiento lógico atómico a prueba de fallos lógicos en la capa de la aplicación.

---

> [!NOTE]
> ### 🏢 El Rascacielos Compartido con Departamentos Aislados
> 
> Entendamos la arquitectura Multi-Tenant unificada con una analogía física e intuitiva:
> 
> - **El Enfoque de Base de Datos Compartida con RLS (El Rascacielos Residencial Unificado)**:
>   - Imagina que decides construir un negocio de vivienda masiva en una gran metrópolis.
>   - En lugar de comprar 100 terrenos separados y construir 100 pequeñas casas independientes con sistemas de tuberías y electricidad individuales (**Bases de datos aisladas por cliente**), decides construir un único rascacielos residencial moderno de 100 departamentos (la **Base de Datos Compartida**).
>   - El rascacielos comparte los mismos cimientos físicos en la tierra, la misma acometida eléctrica principal y el mismo personal de seguridad en el vestíbulo (los recursos de CPU, RAM y disco de tu base de datos central). Esto abarata los costos operativos al mínimo absoluto.
>   - **El Peligro**: Si no construyes paredes internas sólidas, los inquilinos podrían caminar libremente por tu edificio y entrar a las salas de sus vecinos a tomar sus pertenencias (inconsistencia lógica de software).
>   - **La Solución**: Cada departamento está equipado con una cerradura electrónica robusta en su puerta de entrada (**Row-Level Security**). El conserje del elevador te entrega una tarjeta al entrar al lobby codificada con tu número de departamento exacto (**`current_tenant_id`**). 
>   - No importa cuántos pasillos camines o cuántas puertas empujes en el edificio; tu tarjeta solo abrirá la puerta de tu departamento. Tienes los beneficios de costes de vivir en una megaestructura unificada, con la privacidad física total de poseer una mansión aislada en la montaña.

---

## 14.2 Implementación Completa: Esquema Físico y Código Backend

A continuación, implementaremos el código completo y unificado para nuestro SaaS Multi-Tenant Financiero. Primero, configuraremos el esquema físico de base de datos con RLS. Después, implementaremos el backend en TypeScript utilizando el pool de PostgreSQL nativo para gestionar la creación de tenants y el procesamiento transaccional de depósitos financieros:

### 1. El Esquema Físico SQL de Producción
### `esquemaSaaSSecure.sql`
```sql
-- Habilitar extensiones de seguridad para UUIDs si es necesario
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";

-- Tabla de Inquilinos (Tenants)
CREATE TABLE tenants (
    id SERIAL PRIMARY KEY,
    nombre_empresa VARCHAR(150) NOT NULL UNIQUE,
    estado VARCHAR(20) NOT NULL DEFAULT 'activo', -- 'activo', 'suspendido'
    creado_at TIMESTAMP DEFAULT NOW()
);

-- Tabla de Cuentas Financieras Compartida
CREATE TABLE cuentas_saas (
    id SERIAL PRIMARY KEY,
    tenant_id INT NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    nombre_titular VARCHAR(100) NOT NULL,
    saldo NUMERIC(15, 2) NOT NULL DEFAULT 0.00,
    creado_at TIMESTAMP DEFAULT NOW()
);

-- Habilitar Row-Level Security de PostgreSQL de forma explícita
ALTER TABLE cuentas_saas ENABLE ROW LEVEL SECURITY;

-- Crear la política de seguridad RLS Multi-Tenant definitiva.
-- Esta política intercepta todo SELECT, INSERT, UPDATE o DELETE y restringe
-- la visibilidad basándose en la variable local de sesión 'app.current_tenant_id'.
CREATE POLICY policy_cuentas_tenant_isolation ON cuentas_saas
    FOR ALL
    USING (tenant_id = NULLIF(current_setting('app.current_tenant_id', true), '')::integer)
    WITH CHECK (tenant_id = NULLIF(current_setting('app.current_tenant_id', true), '')::integer);
```

### 2. El Orquestador Backend en TypeScript
### `servicioSaaS.ts`
```typescript
import { dbPool } from '../clients/dbClient';

export interface Tenant {
  id: number;
  nombreEmpresa: string;
  estado: string;
}

export interface CuentaSaaS {
  id: number;
  tenantId: number;
  nombreTitular: string;
  saldo: number;
}

// 1. Registrar un nuevo inquilino en el sistema (Lógica administrativa global sin RLS)
export async function registrarNuevoTenant(nombreEmpresa: string): Promise<Tenant> {
  const cliente = await dbPool.connect();

  try {
    const sqlInsert = `
      INSERT INTO tenants (nombre_empresa, estado)
      VALUES ($1, 'activo')
      RETURNING id, nombre_empresa AS "nombreEmpresa", estado
    `;
    const resultado = await cliente.query(sqlInsert, [nombreEmpresa]);
    return resultado.rows[0];

  } catch (error: any) {
    console.error('[SaaS-Admin] Error al registrar tenant:', error.message);
    throw error;
  } finally {
    cliente.release();
  }
}

// 2. Crear una nueva cuenta asignada a un tenant activo
export async function crearCuentaCliente(
  tenantId: number,
  nombreTitular: string,
  saldoInicial: number
): Promise<CuentaSaaS> {
  const cliente = await dbPool.connect();

  try {
    await cliente.query('BEGIN');

    // Establecemos la variable de sesión LOCAL de forma transaccional para pasar RLS
    await cliente.query("SELECT set_config('app.current_tenant_id', $1, true)", [tenantId.toString()]);

    const sqlInsert = `
      INSERT INTO cuentas_saas (tenant_id, nombre_titular, saldo)
      VALUES ($1, $2, $3)
      RETURNING id, tenant_id AS "tenantId", nombre_titular AS "nombreTitular", saldo
    `;
    const resultado = await cliente.query(sqlInsert, [tenantId, nombreTitular, saldoInicial]);

    await cliente.query('COMMIT');
    
    const cuenta = resultado.rows[0];
    return {
      id: cuenta.id,
      tenantId: cuenta.tenantId,
      nombreTitular: cuenta.nombreTitular,
      saldo: parseFloat(cuenta.saldo)
    };

  } catch (error: any) {
    await cliente.query('ROLLBACK');
    console.error(`[SaaS-Tenant-${tenantId}] Fallo al crear cuenta del cliente:`, error.message);
    throw error;
  } finally {
    cliente.release();
  }
}

// 3. Procesar un depósito financiero atómico bajo estricto aislamiento RLS
export async function procesarDepositoSaaS(
  tenantId: number,
  cuentaId: number,
  montoDeposito: number
): Promise<{ exito: boolean; mensaje: string; saldoActual?: number }> {
  
  if (montoDeposito <= 0) {
    return { exito: false, mensaje: 'El monto del depósito debe ser mayor a cero.' };
  }

  const cliente = await dbPool.connect();

  try {
    await cliente.query('BEGIN');

    // Inyectamos la identidad del tenant para asegurar que RLS bloquee accesos cruzados
    await cliente.query("SELECT set_config('app.current_tenant_id', $1, true)", [tenantId.toString()]);

    // Intentamos seleccionar la cuenta con bloqueo exclusivo FOR UPDATE para concurrencia.
    // Si la cuenta pertenece a OTRO tenant, la política RLS filtrará el registro física e invisiblemente,
    // provocando que la consulta devuelva 0 filas. El atacante nunca sabrá si la cuenta ajena existe.
    const sqlObtener = `
      SELECT id, saldo 
      FROM cuentas_saas 
      WHERE id = $1 
      FOR UPDATE
    `;
    const resultado = await cliente.query(sqlObtener, [cuentaId]);

    if (resultado.rows.length === 0) {
      throw new Error('La cuenta destino no existe o no tiene permisos de acceso en este tenant.');
    }

    const cuenta = resultado.rows[0];
    const nuevoSaldo = parseFloat(cuenta.saldo) + montoDeposito;

    // Actualizamos el saldo
    const sqlActualizar = `
      UPDATE cuentas_saas 
      SET saldo = $1 
      WHERE id = $2
    `;
    await cliente.query(sqlActualizar, [nuevoSaldo, cuentaId]);

    await cliente.query('COMMIT');
    return { exito: true, mensaje: 'Depósito procesado con éxito.', saldoActual: nuevoSaldo };

  } catch (error: any) {
    await cliente.query('ROLLBACK');
    console.error(`[SaaS-Tenant-${tenantId}] Error en depósito para cuenta ${cuentaId}:`, error.message);
    return { exito: false, mensaje: error.message };
  } finally {
    cliente.release();
  }
}

// 4. Consultar saldos acumulados de forma segura con protección RLS
export async function listarCuentasPorTenant(tenantId: number): Promise<CuentaSaaS[]> {
  const cliente = await dbPool.connect();

  try {
    await cliente.query('BEGIN');
    await cliente.query("SELECT set_config('app.current_tenant_id', $1, true)", [tenantId.toString()]);

    // PostgreSQL inyecta automáticamente la cláusula WHERE tenant_id = app.current_tenant_id
    const sqlListar = `SELECT id, tenant_id AS "tenantId", nombre_titular AS "nombreTitular", saldo FROM cuentas_saas`;
    const resultado = await cliente.query(sqlListar);

    await cliente.query('COMMIT');

    return resultado.rows.map(row => ({
      id: row.id,
      tenantId: row.tenantId,
      nombreTitular: row.nombreTitular,
      saldo: parseFloat(row.saldo)
    }));

  } catch (error: any) {
    await cliente.query('ROLLBACK');
    console.error(`[SaaS-Tenant-${tenantId}] Error al listar cuentas del tenant:`, error.message);
    throw error;
  } finally {
    cliente.release();
  }
}
```

---

## Resumen del Capítulo

* La arquitectura **Single-Database Multi-Tenant** permite optimizar al máximo el uso de hardware, RAM y mantenimiento unificando la infraestructura de todos los clientes en un mismo clúster físico.
* El mecanismo **Row-Level Security (RLS)** erradica el peligro de fallos en el código de la aplicación (como omitir un `WHERE`) inyectando el aislamiento directamente en la capa física de PostgreSQL.
* Configurar variables de sesión locales y temporales (`set_config` con `is_local = true`) dentro de bloques transaccionales es el estándar industrial definitivo para comunicar la identidad del usuario con las políticas RLS de forma atómica y segura.
* Las consultas con protección exclusiva (`FOR UPDATE`) en combinación con políticas RLS garantizan la total integridad ante escrituras concurrentes sobre cuentas de inquilinos individuales.

En el próximo capítulo y apéndice final, consolidaremos nuestra maestría técnica mediante una **Cheat Sheet analítica definitiva y una sección completa de Ejercicios Resueltos Paso a Paso** de todos los niveles de dificultad.

---

[← Capítulo anterior (Capítulo 13)](13-migraciones-zero-downtime.md) | [Inicio (README.md)](README.md) | [Capítulo siguiente (Capítulo 15) →](15-ejercicios-y-cheat-sheet.md)
