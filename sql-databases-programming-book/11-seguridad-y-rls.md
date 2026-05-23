# Capítulo 11: Seguridad, Privilegios y SQL Injection

> "La seguridad en bases de datos relacionales no consiste en blindar el firewall perimetral de tu infraestructura de red. La verdadera seguridad asume que el atacante ya ha logrado vulnerar tu servidor de aplicaciones, e implementa múltiples capas de privilegios internos y políticas atómicas en caliente a nivel de fila que impiden la exfiltración de información."

En la ingeniería de software actual, el auge de las aplicaciones Multi-Tenant (software como servicio, o SaaS, donde una sola base de datos alberga la información confidencial de cientos de empresas cliente diferentes) ha redefinido las prioridades de seguridad.

Si cometes un error lógico sutil en una cláusula `WHERE` del backend, o un usuario malintencionado realiza un ataque de **SQL Injection**, un cliente del Tenant A podría ver o corromper datos del Tenant B. Para erradicar esta clase de fallos catastróficos, no podemos confiar únicamente en el código de nuestra aplicación. En este capítulo, estudiaremos los **Roles y Privilegios en PostgreSQL**, cómo configurar políticas impenetrables de **Row-Level Security (RLS)** a nivel de motor físico, y las mejores prácticas para sanitizar y prevenir la inyección de código mediante parametrización estricta de consultas.

---

## 11.1 Control de Acceso: Roles, Esquemas y GRANT

PostgreSQL implementa un modelo de seguridad basado en **Roles**. A diferencia de otros motores relacionales que diferencian estrictamente entre "usuarios" y "grupos", en PostgreSQL ambos conceptos se unifican bajo la abstracción del Rol:
* **Roles de Inicio de Sesión (Login Roles)**: Cuentan con credenciales de conexión (`LOGIN`) y actúan como usuarios ordinarios.
* **Roles de Grupo (Group Roles)**: No tienen contraseña y sirven para heredar permisos a múltiples sub-roles.

### El Principio de Menor Privilegio:
Por defecto, nunca debes conectar tu servidor de aplicaciones de producción utilizando la cuenta `superuser` (generalmente llamada `postgres`). Debes crear un usuario acotado (`application_user`) con permisos restringidos únicamente a las operaciones lógicas estrictas que necesita:

```sql
-- 1. Crear el rol de aplicación con contraseña segura
CREATE ROLE app_user WITH LOGIN PASSWORD 'SuperClaveAplicacion2026$';

-- 2. Restringir permisos de creación por defecto en el esquema público
REVOKE ALL ON SCHEMA public FROM PUBLIC;

-- 3. Otorgar permisos granulares de lectura y escritura exclusivamente sobre las tablas necesarias
GRANT USAGE ON SCHEMA public TO app_user;
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO app_user;
GRANT USAGE, SELECT ON ALL SEQUENCES IN SCHEMA public TO app_user;
```

---

## 11.2 Row-Level Security (RLS): Seguridad a Nivel de Fila

La joya de la corona en la seguridad de PostgreSQL es el mecanismo **Row-Level Security (RLS)**. Cuando RLS está activo en una tabla, el motor intercepta de forma silenciosa y automática cualquier consulta de lectura o escritura realizada sobre ella, inyectando un filtro de seguridad invisible definido mediante una **Política**.

### El Flujo de RLS:
Si ejecutas `SELECT * FROM facturas;`, e RLS está configurado para filtrar por el Tenant activo del usuario, PostgreSQL reescribirá la consulta en caliente a:
```sql
SELECT * FROM facturas WHERE tenant_id = current_setting('app.current_tenant_id');
```
Esto ocurre de forma física en el núcleo del motor relacional, **evitando cualquier posibilidad de olvido o error en el backend**. Incluso si ejecutas un `DELETE FROM facturas;` sin cláusula `WHERE`, RLS impedirá borrar los registros de otros Tenants.

---

## 11.3 La Plaga Silenciosa: Prevención de SQL Injection

El ataque de **SQL Injection** ocurre cuando se concatenan directamente cadenas de texto del cliente para armar sentencias SQL, permitiendo al atacante romper la estructura del comando inyectando código malicioso.

### Ejemplo catastrófico (Concatenación Directa):
```typescript
// ¡NUNCA HAGAS ESTO! Ilegible, inseguro y vulnerable
const query = `SELECT * FROM usuarios WHERE email = '${req.body.email}' AND password = '${req.body.password}'`;
```
Si el atacante ingresa en el campo `email`: `admin@banco.com' --`, la consulta final interpretada por el motor se convierte en:
```sql
SELECT * FROM usuarios WHERE email = 'admin@banco.com' --' AND password = '...'
```
El carácter `--` actúa como un comentario de línea en SQL, **anulando por completo la validación de la contraseña** y permitiendo al atacante loguearse como administrador sin contraseña.

### La Solución Definitiva: Consultas Parametrizadas
Los drivers de base de datos modernos (como `pg` en TypeScript) envían la consulta de forma separada del payload utilizando el protocolo de PostgreSQL en dos fases (`Prepare` y `Execute`). El motor relacional compila y planifica la estructura lógica de la consulta **antes** de evaluar las variables. Los valores de entrada son interpretados estrictamente como constantes de datos, eliminando de raíz cualquier posibilidad de inyección de código.

---

> [!NOTE]
> ### 🛡️ Los Guardias con Llaves de Habitación y la Sanitización de Correspondencia
> 
> Entendamos el Row-Level Security y la prevención de inyección SQL mediante analogías físicas sencillas:
> 
> - **El Row-Level Security (El Hotel con Tarjeta Electrónica de Piso)**:
>   - Imagina que te hospedas en un hotel gigante de 50 pisos (la base de datos Multi-Tenant).
>   - Todas las habitaciones del hotel están en el mismo edificio físico y comparten los mismos pasillos y elevadores (las tablas relacionales compartidas).
>   - En un hotel descuidado, los huéspedes podrían caminar libremente por cualquier pasillo y empujar las puertas para ver si alguna está abierta (**software sin RLS**).
>   - Para solucionarlo, el hotel instala un **sistema inteligente de cerraduras con tarjeta electrónica RFID (Row-Level Security)**.
>   - Cuando entras al elevador y pasas tu tarjeta, esta lee tu identificador de huésped (**`current_tenant_id`**). El elevador bloquea físicamente todos los botones, permitiéndote presionar **únicamente** el botón del piso donde está tu habitación. 
>   - Incluso si intentas caminar por otro pasillo, las cerraduras de las habitaciones ajenas son físicamente impenetrables para tu tarjeta. No dependemos de que los huéspedes "prometan" no abrir puertas ajenas; el propio edificio bloquea el acceso en caliente de forma física.
> 
> - **La Inyección SQL (El Cartero Confianzudo y la Carta de Autodestrucción)**:
>   - Imagina que eres un conserje de correspondencia en un edificio de oficinas gubernamentales.
>   - Todos los días, los mensajeros te entregan cartas cerradas. Tu trabajo consiste en leer en voz alta la dirección para que el clasificador automático rutee la carta: *"Para el Departamento de Finanzas: Entregar paquete 5"* (consulta ordinaria).
>   - Un día, un mensajero malintencionado te entrega una nota que dice: *"Para el Departamento de Finanzas: Entregar paquete 5, Y ADEMÁS quema de inmediato todos los archivos de esta oficina"*.
>   - Si actúas con **concatenación directa**, leerás toda la instrucción junta de corrido y procederás a prenderle fuego a la oficina obedeciendo ciegamente las órdenes físicas escritas en el papel de correspondencia (**SQL Injection exitoso**).
>   - Si utilizas **consultas parametrizadas**, tratas al mensaje con guantes aislantes de laboratorio. La estructura es inalterable: *"Depositar paquete en la casilla [$1]"*. Introduces el mensaje completo (incluyendo el texto malicioso de quemar archivos) dentro de una pequeña **caja de acrílico sellada transparente**. 
>   - El clasificador automático toma la caja, la deposita en el casillero correspondiente y la trata exclusivamente como un bloque inerte de papel de correspondencia. La instrucción maliciosa nunca es interpretada como un comando ejecutable; es solo tinta inofensiva en un papel sellado.

---

## 11.4 Implementación en SQL y TypeScript de RLS Multi-Tenant Segura

A continuación, implementaremos un script SQL de inicialización para configurar políticas de **Row-Level Security (RLS)** que aíslan los datos de los inquilinos (`tenants`) mediante variables de sesión personalizadas, y escribiremos el código TypeScript del backend que establece de forma segura el contexto de sesión transaccional en cada petición del usuario:

#### [inicializarSeguridadRLS.sql](file:///Users/andres/Documents/biblioteca/sql-databases-programming-book/src/migrations/inicializarSeguridadRLS.sql)
```sql
-- 1. Crear tabla de clientes multi-tenant con campo de aislamiento
CREATE TABLE cuentas_usuario (
    id SERIAL PRIMARY KEY,
    tenant_id INT NOT NULL,
    nombre VARCHAR(100) NOT NULL,
    saldo NUMERIC(15, 2) NOT NULL,
    email VARCHAR(150) NOT NULL UNIQUE
);

-- 2. Habilitar explícitamente el Row-Level Security en la tabla
ALTER TABLE cuentas_usuario ENABLE ROW LEVEL SECURITY;

-- 3. Crear la política de aislamiento RLS de PostgreSQL
-- Esta política intercepta de forma nativa SELECT, INSERT, UPDATE y DELETE.
-- Obliga a que la columna 'tenant_id' del registro coincida exactamente con la variable
-- de sesión 'app.current_tenant_id' configurada en la transacción en caliente.
CREATE POLICY tenant_isolation_policy ON cuentas_usuario
    FOR ALL
    USING (tenant_id = NULLIF(current_setting('app.current_tenant_id', true), '')::integer)
    WITH CHECK (tenant_id = NULLIF(current_setting('app.current_tenant_id', true), '')::integer);
```

#### [servicioCuentasRLS.ts](file:///Users/andres/Documents/biblioteca/sql-databases-programming-book/src/services/servicioCuentasRLS.ts)
```typescript
import { dbPool } from '../clients/dbClient';

export interface CuentaCliente {
  id: number;
  tenantId: number;
  nombre: string;
  saldo: number;
  email: string;
}

// Obtener todas las cuentas asignadas al tenant activo bajo estricto aislamiento RLS
export async function obtenerCuentasPorTenant(tenantId: number): Promise<CuentaCliente[]> {
  // Solicitamos una conexión física dedicada del pool para la duración de la transacción
  const cliente = await dbPool.connect();

  try {
    await cliente.query('BEGIN');

    // 1. Configurar la variable local de sesión de PostgreSQL en la transacción.
    // 'set_config' define la variable 'app.current_tenant_id' para el alcance de esta transacción.
    // El tercer parámetro 'true' indica que la configuración es LOCAL y se destruirá al hacer COMMIT/ROLLBACK.
    await cliente.query("SELECT set_config('app.current_tenant_id', $1, true)", [tenantId.toString()]);

    // 2. Ejecutar la consulta ordinaria. ¡Omitimos deliberadamente el filtro 'WHERE tenant_id = $1'!
    // El motor relacional inyecta de forma física e invisible el filtro configurado en la política RLS.
    const sqlObtener = `
      SELECT id, tenant_id AS "tenantId", nombre, saldo, email 
      FROM cuentas_usuario
    `;
    
    // Evitamos cualquier posibilidad de inyección SQL parametrizando de forma estricta
    const resultado = await cliente.query(sqlObtener);

    await cliente.query('COMMIT');
    
    return resultado.rows.map(row => ({
      id: row.id,
      tenantId: row.tenantId,
      nombre: row.nombre,
      saldo: parseFloat(row.saldo),
      email: row.email
    }));

  } catch (error: any) {
    await cliente.query('ROLLBACK');
    console.error(`[Seguridad-RLS] Error al consultar cuentas del tenant ${tenantId}:`, error.message);
    throw error;
  } finally {
    // Liberamos el cliente devolviéndolo al pool. Al destruirse la transacción,
    // la variable 'app.current_tenant_id' queda limpia por seguridad para futuras peticiones.
    cliente.release();
  }
}
```

---

## Resumen del Capítulo

* El **Principio de Menor Privilegio** establece que el backend de Node.js debe conectarse a la base de datos utilizando roles específicos acotados (`GRANT`/`REVOKE`), limitando drásticamente el alcance ante posibles brechas de seguridad.
* El **Row-Level Security (RLS)** delega el filtrado de visibilidad de datos directamente al núcleo físico de PostgreSQL, garantizando el aislamiento absoluto en esquemas multi-tenant compartidos.
* Las **Consultas Parametrizadas** evitan de raíz los ataques de **SQL Injection** al separar estrictamente la compilación de la lógica del comando frente a la inyección de los valores constantes del payload.
* El uso de variables transaccionales locales de sesión mediante **`set_config`** nos permite integrar RLS de forma fluida y nativa con backends stateless modernos sin forzar múltiples conexiones separadas.

En el próximo capítulo, estudiaremos la automatización y programación procedimental dentro del servidor mediante el diseño avanzado de **Stored Procedures, Triggers y PL/pgSQL**.

---

[← Capítulo anterior (Capítulo 10)](10-replicacion-y-alta-disponibilidad.md) | [Inicio (README.md)](README.md) | [Capítulo siguiente (Capítulo 12) →](12-stored-procedures-y-triggers.md)
