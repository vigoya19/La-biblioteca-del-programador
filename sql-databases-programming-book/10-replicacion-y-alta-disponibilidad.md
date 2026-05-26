# Capítulo 10: Replicación y Alta Disponibilidad

> "El almacenamiento en un solo servidor de base de datos es un punto único de fallo inaceptable en ingeniería moderna. La verdadera disponibilidad no es la esperanza de que tu servidor nunca se rompa; es el diseño matemático e infraestructural para garantizar que tu sistema siga funcionando sin perder un solo byte de información cuando el servidor principal explote físicamente."

Hasta ahora hemos estudiado la optimización, el modelado y el particionamiento de datos en un entorno lógicamente autónomo. Sin embargo, en sistemas listos para producción empresarial, dependemos de la redundancia y de la resiliencia geográfica. 

Un pico de tráfico masivo puede saturar los recursos de lectura de un solo servidor, o una falla en el hardware de red del proveedor de nube puede desconectar el disco duro principal. En este capítulo, estudiaremos cómo lograr la **Alta Disponibilidad (HA)** mediante **Replicación Física y Lógica**, el impacto del **Replication Lag (Retraso de Réplica)**, el rol de **PgBouncer** como Connection Pooler para evitar desbordar los hilos del sistema operativo, y cómo implementar ruteo dinámico de lectura/escritura en TypeScript.

---

## 10.1 Replicación Relacional: Física vs. Lógica

La replicación consiste en copiar de forma continua y automática todos los cambios realizados en una base de datos principal (**Primary o Maestro**) hacia uno o varios servidores secundarios (**Replica o Esclavo**).

Existen dos aproximaciones arquitectónicas radicalmente distintas:

### 1. Replicación Física (Streaming Replication)
* **Cómo funciona**: Copia a nivel microscópico bit a bit de los archivos del disco duro mediante el flujo secuencial de logs de transacción **Write-Ahead Logging (WAL)** que estudiamos en el Capítulo 3. La réplica es un clon físico exacto idéntico al Primary en todo momento.
* **Pros**: Extremadamente rápida, bajo consumo de CPU en la réplica, y de muy alta fidelidad. Es el pilar de los sistemas de recuperación ante desastres (Disaster Recovery).
* **Contras**: La réplica física es de **sólo lectura** (`Read-Only Replica`). Además, no se puede replicar parcialmente; es todo el servidor físico de base de datos o nada.

### 2. Replicación Lógica (Logical Replication)
* **Cómo funciona**: Transmite los cambios lógicos del sistema (sentencias SQL interpretadas: *"Se insertó la fila con ID=5 en la tabla X"*) utilizando un modelo de **Publicación y Suscripción (Publish/Subscribe)**.
* **Pros**: Permite replicar de forma granular y selectiva (ej: solo replicar la tabla `usuarios` o filtrar registros geográficamente). Las réplicas lógicas pueden ser editables y albergar estructuras de tablas diferentes.
* **Contras**: Mayor consumo de CPU debido al parsing y decodificación de logs en red, y mayor propensión a colisiones lógicas.

---

## 10.2 Replicación Síncrona vs. Asíncrona y Replication Lag

¿Cuándo considera el servidor Primary que una transacción se ha completado con éxito absoluto?

### 1. Replicación Asíncrona (Predeterminada)
El Primary ejecuta el commit en su disco local, responde `OK` inmediatamente al cliente y luego, en background, transmite los cambios en red a las réplicas.
* **Ventaja**: Máxima velocidad de escritura en el cliente.
* **Peligro**: Si el servidor Primary explota antes de transmitir las últimas transacciones, esos bytes se perderán para siempre (**Data Loss**). Además, las réplicas pueden tardar milisegundos o segundos en reflejar la realidad del Primary (**Replication Lag**).

### 2. Replicación Síncrona
El Primary ejecuta la transacción localmente pero **congela la respuesta al cliente** hasta que al menos una réplica secundaria confirme por red que ha recibido y escrito los bytes del WAL en su propio disco duro.
* **Ventaja**: Garantía absoluta de cero pérdida de datos. Si el Primary cae, la réplica tiene exactamente los mismos bytes consolidados.
* **Peligro**: Si la red entre servidores es lenta o una réplica se cuelga, todas las escrituras de tu backend se bloquearán por completo en un estado de espera infinita, degradando la latencia del sistema.

---

## 10.3 Connection Pooling a Gran Escala: PgBouncer

En PostgreSQL, **cada conexión de red entrante desde tu backend abre un hilo de proceso físico independiente (`postgres backend process`)** en el sistema de archivos del servidor. Estos hilos consumen alrededor de 10MB de memoria RAM base de forma inmediata y compiten ferozmente por la CPU. 

Si tu clúster de Node.js en Kubernetes escala a 100 instancias y cada una abre un pool con un máximo de 20 conexiones, tu servidor de base de datos tendrá que gestionar **2,000 conexiones concurrentes**. Esto paralizará por completo al motor debido al coste de context-switch de la CPU del sistema operativo.

### La solución: PgBouncer
PgBouncer es un proxy intermedio ultraligero que realiza **Connection Pooling externo**:
* Tu backend abre miles de conexiones contra PgBouncer.
* PgBouncer las mantiene en cola de forma extremadamente barata en memoria RAM y las multiplexa utilizando un pool real muy pequeño y óptimo directo con PostgreSQL (por ejemplo, solo 50 conexiones físicas reales).
* **Modo de transacción (Transaction Pooling)**: PgBouncer asigna una conexión física al cliente únicamente por la duración exacta de su transacción SQL en caliente. Al ejecutarse el commit, la conexión física se libera y se entrega de inmediato a otro cliente en milisegundos, multiplicando la escalabilidad del sistema por mil.

---

> [!NOTE]
> ### 📠 La Copia de Fax en Tiempo Real y el Conserje del Teléfono
> 
> Entendamos la replicación, el lag y el pool de conexiones con analogías físicas cotidianas:
> 
> - **La Replicación Física (La Copia por Fax de Bitácora)**:
>   - Imagina que eres el Capitán de un barco de carga gigante (el **Primary**). Llevas una bitácora física detallada donde apuntas cada movimiento del timón y del viento.
>   - Tienes a un oficial de comunicaciones al lado que escanea instantáneamente cada página escrita y la manda por fax a la base militar en tierra firme (**Réplica de Lectura**).
>   - La base militar tiene una copia exacta idéntica página por página del libro original (clonación física). Los sargentos en tierra pueden consultar la bitácora todo el día para analizar el trayecto, pero tienen prohibido escribir notas encima; solo el Capitán en alta mar puede editar la bitácora original.
> 
> - **El Replication Lag (La Llamada con Interferencia en el Mar)**:
>   - El barco navega a través de una tormenta eléctrica masiva que genera interferencia de radio.
>   - Escribes un mensaje crítico en la bitácora a las 10:00 AM, pero debido a la mala señal, el fax tarda 5 minutos completos en imprimirse en la base militar (**Replication Lag**).
>   - Si un oficial en tierra consulta la bitácora a las 10:02 AM para saber la ubicación del barco, verá la información de las 09:57 AM. Tomará decisiones basadas en una versión retrasada de la realidad.
> 
> - **PgBouncer (El Conserje de la Cabina Telefónica de Emergencias)**:
>   - Imagina que un hospital tiene una sola cabina de teléfono directo con el centro nacional de salud (el **Servidor de Base de Datos**).
>   - Si 500 médicos intentan entrar físicamente a la cabina al mismo tiempo a hacer llamadas individuales largas, se aplastarán unos a otros rompiendo la cabina y saturando la línea (**Caos de Conexiones Directas**).
>   - El hospital contrata a un **Conserje entrenado** en la puerta (**PgBouncer**). 
>   - El conserje detiene a los 500 médicos de forma ordenada en el pasillo. A medida que un médico necesita reportar una urgencia, el conserje toma el auricular, le permite hablar por 5 segundos para pasar la instrucción analítica transaccional, cuelga y le entrega el auricular al siguiente médico de la fila en menos de un parpadeo. 
>   - El teléfono físico nunca se satura, y todos los médicos logran comunicarse sin colapsar las paredes del hospital.

---

## 10.4 Implementación en TypeScript de Ruteo Dinámico Primary-Replica

A continuación, implementaremos un despachador de base de datos en TypeScript para producción que cuenta con **dos pools de conexiones separados**: uno dirigido al nodo **Primary** (para transacciones de escritura y consultas de alta consistencia crítica) y otro dirigido al balanceador de carga de las réplicas de sólo lectura **Read Replicas**. Rutearemos las consultas de forma inteligente basándonos en la naturaleza de la operación:

### `clienteBaseDeDatosRuteado.ts`
```typescript
import { Pool, PoolClient } from 'pg';

// Configuración de infraestructura ruteada (en producción se lee de variables de entorno)
const CONFIG_PRIMARY = {
  host: 'primary-db.internal.net',
  port: 5432,
  user: 'admin_finanzas',
  password: 'SecurePasswordPrimary99$',
  database: 'banco_core',
  max: 20 // Pool físico acotado
};

const CONFIG_REPLICA = {
  host: 'replicas-lb.internal.net', // Balanceador de réplicas de sólo lectura
  port: 5432,
  user: 'readonly_analisis',
  password: 'SecurePasswordReplica00$',
  database: 'banco_core',
  max: 50 // Mayor tamaño de pool para soportar consultas analíticas
};

class ClientedeBaseDatosRuteado {
  private poolPrimary: Pool;
  private poolReplica: Pool;

  constructor() {
    this.poolPrimary = new Pool(CONFIG_PRIMARY);
    this.poolReplica = new Pool(CONFIG_REPLICA);

    this.poolPrimary.on('error', (err) => console.error('[DB-Primary] Error catastrófico en pool:', err.message));
    this.poolReplica.on('error', (err) => console.error('[DB-Replica] Error catastrófico en pool:', err.message));
  }

  // 1. Obtener un cliente directo del Primary para ejecutar transacciones complejas escritas
  public async obtenerClienteTransaccional(): Promise<PoolClient> {
    const cliente = await this.poolPrimary.connect();
    return cliente;
  }

  // 2. Ejecutar consultas genéricas ruteando dinámicamente según lectura/escritura
  public async ejecutarConsulta<T = any>(
    sqlText: string,
    params: any[] = [],
    opciones?: { forzarConsistenciaEstricta?: boolean }
  ): Promise<T[]> {
    
    // Limpiamos espacios y pasamos a mayúsculas para analizar la naturaleza del SQL
    const consultaNormalizada = sqlText.trim().toUpperCase();
    const esEscritura = 
      consultaNormalizada.startsWith('INSERT') || 
      consultaNormalizada.startsWith('UPDATE') || 
      consultaNormalizada.startsWith('DELETE') || 
      consultaNormalizada.startsWith('ALTER') || 
      consultaNormalizada.startsWith('CREATE') || 
      consultaNormalizada.startsWith('DROP');

    // Criterio de ruteo:
    // Si es una escritura o el usuario pide explícitamente consistencia total (ej. ver saldo justo después de depositar)
    // mandamos la consulta directo al Primary. De lo contrario, ruteamos a Réplicas para balancear la carga.
    if (esEscritura || opciones?.forzarConsistenciaEstricta) {
      console.log('[Router-DB] Ruteando consulta de alta consistencia al clúster PRIMARY.');
      const resultado = await this.poolPrimary.query(sqlText, params);
      return resultado.rows;
    } else {
      console.log('[Router-DB] Ruteando consulta analítica/lectura al clúster READ-REPLICA.');
      try {
        const resultado = await this.poolReplica.query(sqlText, params);
        return resultado.rows;
      } catch (error: any) {
        console.warn(`[Router-DB] Fallo al consultar en réplica: ${error.message}. Ejecutando Fallback automático al PRIMARY.`);
        // Fallback defensivo: Si la réplica falla en red, intentamos rescatar la lectura en el Primary
        const resultado = await this.poolPrimary.query(sqlText, params);
        return resultado.rows;
      }
    }
  }

  // Cerrar pools al apagar el servidor de aplicaciones
  public async apagarConexiones(): Promise<void> {
    await Promise.all([
      this.poolPrimary.end(),
      this.poolReplica.end()
    ]);
    console.log('[Router-DB] Pools de base de datos desconectados de forma limpia.');
  }
}

// Exportamos singleton listo para ser importado por el backend
export const dbRouter = new ClientedeBaseDatosRuteado();
```

---

## Resumen del Capítulo

* La **Replicación Física (WAL streaming)** ofrece clones de disco de sólo lectura sumamente eficientes y con un mínimo impacto de CPU. La **Replicación Lógica** permite sincronizaciones selectivas, personalizadas y modificables bajo un modelo Publish/Subscribe.
* La **Replicación Asíncrona** destaca en latencia pero está sujeta a la posibilidad de pérdidas de datos si ocurre un desastre repentino. La **Replicación Síncrona** garantiza consistencia a cambio de incrementar la latencia de las escrituras del sistema.
* **PgBouncer** optimiza el uso de CPU y memoria RAM en servidores PostgreSQL relacionales mediante la multiplexación inteligente de miles de conexiones virtuales sobre un pool acotado de conexiones físicas reales.
* El ruteo de lectura/escritura del lado de la aplicación nos permite maximizar el uso de hardware, descargando la inmensa mayoría de consultas analíticas pesadas sobre réplicas balanceadas de lectura.

En el próximo capítulo, abordaremos los mecanismos definitivos de protección de datos en la base de datos mediante el estudio de **Seguridad, Privilegios y Row-Level Security (RLS)** para SaaS multi-tenant.

---

[← Capítulo anterior (Capítulo 9)](09-particionamiento-y-sharding.md) | [Inicio (README.md)](README.md) | [Capítulo siguiente (Capítulo 11) →](11-seguridad-y-rls.md)
