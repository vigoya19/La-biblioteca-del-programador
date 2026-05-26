# Capítulo 9: JDBC — Acceso a Bases de Datos en Java

---

## 9.1 ¿Qué es JDBC?

JDBC (*Java Database Connectivity*) es la API estándar de Java para conectarse a bases de datos relacionales. Forma parte de la plataforma Java SE (paquete `java.sql` y `javax.sql`) y permite ejecutar sentencias SQL desde aplicaciones Java de forma independiente del motor de base de datos subyacente. JDBC actúa como una capa de abstracción: escribes el mismo código Java para PostgreSQL, MySQL, Oracle o H2, y el driver específico traduce las llamadas al protocolo de cada motor.

### ¿Por qué aprender JDBC si existen frameworks como Hibernate o JPA?

Entender JDBC te permite:

- **Comprender qué ocurre bajo el capó** de cualquier framework ORM o de acceso a datos.
- **Depurar problemas de rendimiento**: saber qué SQL se está ejecutando realmente.
- **Escribir consultas complejas** que un ORM no puede generar eficientemente.
- **Controlar exactamente la interacción** con la base de datos sin "magia".
- **Optimizar operaciones masivas** (batch, bulk, ETL) donde JDBC directo supera ampliamente a un ORM.

### Arquitectura

La arquitectura de JDBC se compone de los siguientes elementos:

```
Aplicación Java
     │
     ▼
DriverManager ──▶ Driver ──▶ Base de Datos
     │
     ▼
Connection ──▶ Statement ──▶ ResultSet
```

- **`DriverManager`**: Punto de entrada. Gestiona la lista de drivers disponibles y selecciona el adecuado según la URL de conexión.
- **`Driver`**: Implementación específica para cada base de datos. Traduce las llamadas JDBC al protocolo nativo del motor.
- **`Connection`**: Representa una sesión con la base de datos. Proporciona métodos para crear statements y controlar transacciones.
- **`Statement`**: Objeto que envía sentencias SQL a la base de datos.
- **`ResultSet`**: Conjunto de resultados devuelto por una consulta `SELECT`.

### Flujo de trabajo típico

```
1. Registrar Driver (automático desde Java 6)
2. Obtener Connection (DriverManager o DataSource)
3. Crear Statement / PreparedStatement
4. Ejecutar SQL (executeQuery / executeUpdate / execute)
5. Procesar ResultSet (si es SELECT)
6. Cerrar recursos (try-with-resources)
```

### Drivers más comunes

| Base de datos | Clase del Driver | Artefacto Maven |
|---|---|---|
| MySQL | `com.mysql.cj.jdbc.Driver` | `mysql-connector-j` |
| PostgreSQL | `org.postgresql.Driver` | `postgresql` |
| H2 (embedded) | `org.h2.Driver` | `h2` |
| Oracle | `oracle.jdbc.OracleDriver` | `ojdbc8` |
| SQL Server | `com.microsoft.sqlserver.jdbc.SQLServerDriver` | `mssql-jdbc` |
| SQLite | `org.sqlite.JDBC` | `sqlite-jdbc` |
| MariaDB | `org.mariadb.jdbc.Driver` | `mariadb-java-client` |

### Registro automático de drivers

Desde Java 6, el mecanismo `ServiceLoader` registra los drivers automáticamente. Basta con incluir el JAR del driver en el classpath. La clase del driver contiene un bloque estático que se registra ante `DriverManager`:

```java
// En el código del driver (ej. MySQL):
static {
    try {
        DriverManager.registerDriver(new com.mysql.cj.jdbc.Driver());
    } catch (SQLException e) {
        throw new RuntimeException("Can't register driver!");
    }
}
```

El programador **no necesita** llamar a `Class.forName()` explícitamente (práctica obsoleta anterior a Java 6).

---

## 9.2 Conexión a bases de datos

### Establecer una conexión

El método `DriverManager.getConnection()` acepta una URL JDBC y, opcionalmente, usuario y contraseña:

```java
Connection conn = DriverManager.getConnection(url, usuario, password);
// o sin credenciales para bases como H2 en modo embedded
Connection conn = DriverManager.getConnection(url);
```

### Formato de URL JDBC

La URL JDBC sigue el formato `jdbc:<subprotocolo>://<host>:<puerto>/<base>`. Ejemplos:

```
jdbc:mysql://localhost:3306/mi_base?useSSL=false&serverTimezone=UTC
jdbc:postgresql://localhost:5432/mi_base
jdbc:postgresql://localhost:5432/mi_base?currentSchema=public
jdbc:h2:~/data/mi_base          // archivo en disco
jdbc:h2:mem:mi_base             // en memoria (testing)
jdbc:h2:mem:                    // base anónima en memoria
jdbc:mariadb://localhost:3307/mi_base
jdbc:sqlserver://localhost:1433;databaseName=mi_base
jdbc:oracle:thin:@localhost:1521:XE
```

**Propiedades adicionales** pueden pasarse por URL (como parámetros query) o mediante un objeto `Properties`:

```java
Properties props = new Properties();
props.setProperty("user", "admin");
props.setProperty("password", "secreto");
props.setProperty("useSSL", "false");
props.setProperty("serverTimezone", "UTC");
Connection conn = DriverManager.getConnection("jdbc:mysql://localhost:3306/mi_base", props);
```

### Ciclo de vida de una conexión

Toda conexión debe seguir el ciclo: **abrir → usar → cerrar**. Una conexión no cerrada consume recursos del servidor de base de datos y puede agotar el pool.

**Siempre usar try-with-resources** (desde Java 7) para garantizar el cierre automático:

```java
try (Connection conn = DriverManager.getConnection(url, user, pass)) {
    // usar la conexión
} catch (SQLException e) {
    e.printStackTrace();
}
// Connection se cierra automáticamente al salir del bloque try
```

**Diagrama de estados de una conexión:**

```
     ┌─────────┐
     │ CLOSED  │ ◀──────────────────────────────────────┐
     └────┬────┘                                        │
          │ open                                        │
          ▼                                             │
     ┌─────────┐     setAutoCommit(false)    ┌──────────┴───┐
     │  IDLE   │ ───────────────────────────▶│ IN_TRANSACTION│
     └─────────┘                              └──────┬───────┘
          ▲                                          │
          │               commit/rollback            │
          └──────────────────────────────────────────┘
```

### Introduction al Connection Pool

Crear una conexión nueva para cada petición es costoso (handshake TCP, autenticación, asignación de recursos en el servidor). Un **pool de conexiones** mantiene un conjunto de conexiones abiertas y reutilizables.

```
Sin pool:
Petición 1 → abrir → usar → cerrar
Petición 2 → abrir → usar → cerrar
Petición 3 → abrir → usar → cerrar
(Tiempo: 30ms por apertura = 90ms solo en conexiones)

Con pool:
Inicialización → [conn1] [conn2] [conn3] [conn4] [conn5]
Petición 1 → tomar conn1 → usar → devolver
Petición 2 → tomar conn2 → usar → devolver
Petición 3 → tomar conn3 → usar → devolver
(Tiempo: ~0ms en obtención de conexión)
```

**HikariCP** es el pool más rápido y más usado en el ecosistema Java:

```xml
<!-- Maven -->
<dependency>
    <groupId>com.zaxxer</groupId>
    <artifactId>HikariCP</artifactId>
    <version>5.1.0</version>
</dependency>
```

```java
HikariConfig config = new HikariConfig();
config.setJdbcUrl("jdbc:mysql://localhost:3306/mi_base");
config.setUsername("admin");
config.setPassword("secreto");
config.setMaximumPoolSize(10);
config.setMinimumIdle(5);
HikariDataSource ds = new HikariDataSource(config);

try (Connection conn = ds.getConnection()) {
    // trabajar con la conexión
}
```

---

## 9.3 Connection Pool en profundidad

### ¿Por qué HikariCP es el más rápido?

HikariCP es el pool de conexiones por defecto en Spring Boot 2+ y es reconocido como el más rápido del ecosistema Java. Sus optimizaciones incluyen:

1. **Bytecode-level engineering**: HikariCP genera bytecode optimizado en tiempo de ejecución para sus proxies de `Connection`, `Statement` y `ResultSet`, evitando reflexión y reduciendo la sobrecarga de cada llamada.

2. **Mínima contención de locks**: usa estructuras de datos lock-free (`ConcurrentBag`) para gestionar las conexiones disponibles. No hay bloqueos globales al pedir o devolver conexiones.

3. **Validación de conexiones sin sobrecarga**: en lugar de ejecutar una query de prueba (`SELECT 1`), HikariCP puede validar conexiones verificando la marca de tiempo de la última comunicación, mucho más rápido cuando el driver soporta JDBC 4.

4. **Optimización de Timeout**: usa `ScheduledExecutorService` con precisión de milisegundos para detectar timeouts, sin crear un hilo por timeout.

5. **JIT-friendly**: el código está diseñado para que el compilador JIT de la JVM pueda optimizarlo agresivamente (pocas ramas, pocas indirecciones).

6. **Zero-overhead en estado estacionario**: cuando el pool está estable (sin crear ni destruir conexiones), el coste de obtener y devolver una conexión es prácticamente nulo.

### Configuración de HikariCP en detalle

Cada parámetro de configuración tiene un impacto directo en el rendimiento y la estabilidad:

```java
HikariConfig config = new HikariConfig();

// --- Configuración de conexión ---
config.setJdbcUrl("jdbc:postgresql://db-produccion.internal:5432/mi_app");
config.setUsername(System.getenv("DB_USERNAME"));
config.setPassword(System.getenv("DB_PASSWORD"));

// --- Dimensionamiento del pool ---
config.setMaximumPoolSize(20);       // máximo de conexiones en el pool
config.setMinimumIdle(5);           // conexiones inactivas mínimas a mantener

// --- Timeouts ---
config.setConnectionTimeout(30_000);  // ms máximos esperando una conexión del pool
config.setIdleTimeout(600_000);       // ms antes de eliminar una conexión inactiva (10 min)
config.setMaxLifetime(1_800_000);     // ms de vida máxima de una conexión (30 min)
config.setValidationTimeout(5_000);   // ms máximos para validar una conexión

// --- Detección de leaks ---
config.setLeakDetectionThreshold(60_000); // registrar advertencia si conexión no se devuelve

// --- Rendimiento ---
config.setAutoCommit(true);
config.setPoolName("MiApp-Pool");

// --- Validación ---
config.setConnectionTestQuery("SELECT 1");

HikariDataSource ds = new HikariDataSource(config);
```

### Parámetros explicados

| Parámetro | Default | Descripción |
|---|---|---|
| `maximumPoolSize` | 10 | Conexiones máximas (activas + inactivas). Parámetro **más crítico**. |
| `minimumIdle` | = maximumPoolSize | Conexiones inactivas mínimas. HikariCP recomienda igualarlo a maximumPoolSize. |
| `connectionTimeout` | 30000 ms | Tiempo máximo que un hilo espera para obtener una conexión. Si se agota, lanza `SQLException`. |
| `idleTimeout` | 600000 ms | Tiempo máximo que una conexión puede estar inactiva antes de ser eliminada. |
| `maxLifetime` | 1800000 ms | Tiempo máximo de vida de una conexión. Debe ser menor que el timeout del servidor de BD. |
| `leakDetectionThreshold` | 0 (desactivado) | Si > 0, registra advertencia con stack trace cuando conexión no se devuelve. |
| `validationTimeout` | 5000 ms | Tiempo máximo para validar una conexión. |
| `connectionTestQuery` | null | Query ligera a ejecutar antes de entregar una conexión. `SELECT 1` es lo habitual. |

### Fórmula para calcular el tamaño del pool

Existe una fórmula ampliamente aceptada para dimensionar el pool de conexiones:

```
connections = ((core_count * 2) + effective_spindle_count)
```

Donde:
- **core_count**: número de núcleos de CPU disponibles para la aplicación.
- **effective_spindle_count**: número de discos (spindles) que la BD usa para I/O. En SSD modernos o cloud, suele ser 1. Si usas SAN con múltiples discos, indícalo.

**Ejemplo:** Un servidor de 4 núcleos con SSD:

```
connections = ((4 * 2) + 1) = 9
```

Un pool de 9 conexiones suele ser más que suficiente para una aplicación típica. Más conexiones **no mejora** el rendimiento; de hecho, puede degradarlo porque cada conexión activa compite por recursos en la BD.

**Reglas prácticas:**
- Monolito web típico: `maximumPoolSize = 10`
- Microservicio con pocas consultas: `maximumPoolSize = 5`
- Aplicación batch/ETL: `maximumPoolSize = 2-3` (menos concurrencia, más rendimiento por conexión)

### Cómo detectar leaks de conexiones

Un *connection leak* ocurre cuando el código obtiene una conexión del pool pero NUNCA llama a `close()`. Con el tiempo, el pool se agota y la aplicación deja de funcionar.

**Síntoma clásico:** "HikariPool-1 - Connection is not available, request timed out after 30000ms"

**Configuración de detección:**

```java
config.setLeakDetectionThreshold(10_000); // 10s: si una conexión no se devuelve, loggear WARN
```

Ejemplo de log de leak:

```
[HikariPool-1 housekeeper] WARN  c.z.h.p.ProxyLeakTask - Connection leak detection
triggered for conn1 on thread pool-1-thread-3, stack trace follows
java.lang.Exception: Apparent connection leak detected
    at com.zaxxer.hikari.pool.HikariPool.getConnection(HikariPool.java:196)
    at com.mi.paquete.MiServicio.procesar(MiServicio.java:42)
```

**Causa común:** olvidar el try-with-resources:

```java
// MAL: la conexión nunca se cierra → LEAK
Connection conn = ds.getConnection();
PreparedStatement ps = conn.prepareStatement("SELECT ...");
ResultSet rs = ps.executeQuery();
rs.close();
ps.close();
// FALTA conn.close() ← LEAK

// BIEN: try-with-resources lo cierra todo automáticamente
try (Connection conn = ds.getConnection();
     PreparedStatement ps = conn.prepareStatement("SELECT ...");
     ResultSet rs = ps.executeQuery()) {
    // ...
}
```

### Comparativa de Pools de Conexiones

| Característica | HikariCP | Apache DBCP2 | Tomcat JDBC Pool | C3P0 |
|---|---|---|---|---|
| **Velocidad** | ★★★★★ | ★★★☆☆ | ★★★★☆ | ★★☆☆☆ |
| **Uso de memoria** | Mínimo | Medio | Bajo | Alto |
| **Detección de leaks** | Nativa | Via JMX | Via JMX | Manual |
| **Validación conexiones** | JDBC4 + query | Query | Query | Query |
| **Configuración** | Simple | Compleja | Media | Compleja |
| **Soporte JMX** | Sí | Sí | Sí | Sí |
| **PreparedStatement caching** | Delega a driver | Sí | No | Sí |
| **Mantenimiento** | Activo | Activo | Activo | ~Abandonado |
| **Benchmark (ops/s)** | ~1,200,000 | ~500,000 | ~700,000 | ~200,000 |

**Recomendación:** HikariCP para todo proyecto nuevo. Solo considera Tomcat JDBC Pool si ya estás en ecosistema Tomcat sin Spring Boot. Evita C3P0 en proyectos nuevos.

### Configuración completa de HikariCP para producción

```java
import com.zaxxer.hikari.HikariConfig;
import com.zaxxer.hikari.HikariDataSource;

public class DatabaseConfig {

    public static HikariDataSource createProductionDataSource() {
        HikariConfig config = new HikariConfig();

        // --- Conexión desde variables de entorno ---
        config.setJdbcUrl(System.getenv("DB_URL"));
        config.setUsername(System.getenv("DB_USERNAME"));
        config.setPassword(System.getenv("DB_PASSWORD"));

        // --- Pool sizing ---
        config.setMaximumPoolSize(20);
        config.setMinimumIdle(20);  // mantener siempre 20 para evitar creación bajo carga

        // --- Timeouts ---
        config.setConnectionTimeout(30_000);   // 30 segundos esperando conexión
        config.setIdleTimeout(300_000);        // 5 minutos inactiva → eliminar
        config.setMaxLifetime(1_200_000);      // 20 minutos vida máxima
        config.setValidationTimeout(5_000);    // 5 segundos para validar

        // --- Detección de leaks ---
        config.setLeakDetectionThreshold(15_000); // 15s → loggear leak

        // --- Identificación ---
        config.setPoolName("produccion-pool");

        // --- Propiedades del driver (ej. MySQL) ---
        config.addDataSourceProperty("cachePrepStmts", "true");
        config.addDataSourceProperty("prepStmtCacheSize", "250");
        config.addDataSourceProperty("prepStmtCacheSqlLimit", "2048");
        config.addDataSourceProperty("useServerPrepStmts", "true");

        // --- Health check ---
        config.setConnectionTestQuery("SELECT 1");

        HikariDataSource ds = new HikariDataSource(config);

        // --- Graceful shutdown ---
        Runtime.getRuntime().addShutdownHook(new Thread(() -> {
            System.out.println("Cerrando pool de conexiones...");
            ds.close();
            System.out.println("Pool cerrado correctamente.");
        }));

        return ds;
    }
}
```

### Métricas importantes del pool

Monitorizar el pool en producción es esencial. HikariCP expone métricas vía JMX y vía Prometheus:

```java
// Acceso programático a métricas
HikariPoolMXBean poolMXBean = ds.getHikariPoolMXBean();

int activas = poolMXBean.getActiveConnections();
int inactivas = poolMXBean.getIdleConnections();
int totales = poolMXBean.getTotalConnections();
int pendientes = poolMXBean.getThreadsAwaitingConnection();

System.out.printf("Pool [activas=%d, inactivas=%d, total=%d, pendientes=%d]%n",
        activas, inactivas, totales, pendientes);
```

**Métricas a vigilar:**

| Métrica | Significado | Alarma |
|---|---|---|
| `ActiveConnections` | Conexiones en uso ahora mismo | Si se acerca a `maximumPoolSize`, necesitas más pool o hay consultas lentas |
| `IdleConnections` | Conexiones disponibles para uso inmediato | Si es 0 constantemente, el pool está saturado |
| `PendingThreads` | Hilos esperando una conexión | Si > 0, los usuarios experimentan latencia. Crítico si crece |
| `ConnectionTimeoutRate` | % de peticiones que exceden `connectionTimeout` | Si > 0%, la aplicación está fallando |
| `ConnectionCreationRate` | Conexiones creadas por segundo | Si es alto y sostenido, hay leak o pool muy pequeño |

### Pool Sizing en la práctica

**Escenario 1: API REST con 50 peticiones concurrentes**

```
Núcleos de CPU: 8, SSD: 1 disco
Tamaño recomendado: ((8 * 2) + 1) = 17 conexiones

Si el tiempo medio de respuesta de BD es 5ms:
- 17 conexiones → 3,400 consultas/segundo
- Cada petición API hace 2 consultas → 1,700 peticiones/segundo
- Sobrado para 50 concurrentes
```

**Escenario 2: Aplicación batch que procesa archivos**

```
Núcleos de CPU: 4, SSD: 1 disco
Tamaño recomendado: ((4 * 2) + 1) = 9 → reducir a 2-3 conexiones

El cuello de botella es la BD escribiendo, no la concurrencia.
Más conexiones = más contención de locks.
```

---

## 9.4 Statement y PreparedStatement

### Statement (SQL estático)

`Statement` envía sentencias SQL literales a la base de datos. Adecuado para DDL o consultas sin parámetros:

```java
try (Connection conn = DriverManager.getConnection(url, user, pass);
     Statement stmt = conn.createStatement()) {

    stmt.executeUpdate("CREATE TABLE IF NOT EXISTS usuarios (" +
        "id BIGINT AUTO_INCREMENT PRIMARY KEY, " +
        "nombre VARCHAR(100), " +
        "email VARCHAR(100))");

} catch (SQLException e) {
    e.printStackTrace();
}
```

### Inyección SQL — El peligro

**Nunca** construyas consultas concatenando strings con datos del usuario:

```java
// PELIGRO: código vulnerable a SQL injection
String nombre = request.getParameter("nombre"); // usuario envía:  ' OR '1'='1
String sql = "SELECT * FROM usuarios WHERE nombre = '" + nombre + "'";
ResultSet rs = stmt.executeQuery(sql);
// Resultado: SELECT * FROM usuarios WHERE nombre = '' OR '1'='1'
// ¡Devuelve TODOS los registros!
```

Un atacante podría inyectar `'; DROP TABLE usuarios; --` para destruir la tabla por completo.

**Ejemplo real de ataque (formulario de login):**

```java
// CÓDIGO VULNERABLE — NUNCA HACER ESTO
String user = request.getParameter("user"); // atacante: admin' --
String pass = request.getParameter("pass");
String sql = "SELECT * FROM usuarios WHERE username='" + user + "' AND password='" + pass + "'";
// SQL resultante: SELECT * FROM usuarios WHERE username='admin' --' AND password='cualquiera'
// El '--' comenta el resto → se salta la verificación de contraseña
```

### PreparedStatement (consultas parametrizadas)

`PreparedStatement` previene la inyección SQL al separar la estructura de la consulta de los valores:

```java
String sql = "SELECT * FROM usuarios WHERE nombre = ? AND email = ?";

try (Connection conn = DriverManager.getConnection(url, user, pass);
     PreparedStatement ps = conn.prepareStatement(sql)) {

    ps.setString(1, nombre);
    ps.setString(2, email);
    ResultSet rs = ps.executeQuery();

} catch (SQLException e) {
    e.printStackTrace();
}
```

**¿Por qué PreparedStatement es inmune a SQL injection?**

El driver envía la consulta y los valores por separado:
1. Primero envía la estructura SQL con placeholders: `SELECT ... WHERE nombre = ?`
2. La base de datos compila (parsea) el SQL y crea el plan de ejecución.
3. Luego envía los valores por separado como datos, no como parte del SQL.
4. La base de datos usa esos valores para los parámetros del plan ya compilado.

El atacante puede enviar lo que quiera como valor, pero nunca se interpretará como SQL.

### Métodos de asignación de parámetros

| Método | Tipo SQL | Uso |
|---|---|---|
| `setString(int index, String x)` | VARCHAR, CHAR, TEXT | Cadenas de texto |
| `setInt(int index, int x)` | INTEGER | Enteros 32 bits |
| `setLong(int index, long x)` | BIGINT | Enteros 64 bits |
| `setDouble(int index, double x)` | DOUBLE, FLOAT | Decimales |
| `setBigDecimal(int index, BigDecimal x)` | DECIMAL, NUMERIC | Decimales precisos |
| `setBoolean(int index, boolean x)` | BOOLEAN | Booleanos |
| `setDate(int index, Date x)` | DATE | Fecha (sin hora) |
| `setTimestamp(int index, Timestamp x)` | TIMESTAMP | Fecha y hora |
| `setTime(int index, Time x)` | TIME | Hora |
| `setObject(int index, Object x)` | (detecta autom.) | Tipo genérico |
| `setNull(int index, int sqlType)` | NULL | Valor nulo explícito |
| `setBlob(int index, InputStream x)` | BLOB | Datos binarios |
| `setClob(int index, Reader x)` | CLOB | Texto largo |
| `setArray(int index, Array x)` | ARRAY | Arrays SQL |
| `setBytes(int index, byte[] x)` | BINARY, VARBINARY | Datos binarios |

**Nota:** Los índices de los parámetros empiezan en 1 (no en 0).

### Beneficios de PreparedStatement

- **Seguridad**: inmune a SQL injection porque el driver escapa los valores.
- **Rendimiento**: la consulta se pre-compila en la base de datos. Ejecuciones posteriores reutilizan el plan.
- **Claridad**: separa la lógica SQL de los valores, haciendo el código más legible.

### Ejecución por lotes (Batch) — Introducción

Para insertar o actualizar múltiples registros de forma eficiente:

```java
String sql = "INSERT INTO usuarios (nombre, email) VALUES (?, ?)";

try (Connection conn = DriverManager.getConnection(url, user, pass);
     PreparedStatement ps = conn.prepareStatement(sql)) {

    conn.setAutoCommit(false);

    for (Usuario u : listaUsuarios) {
        ps.setString(1, u.getNombre());
        ps.setString(2, u.getEmail());
        ps.addBatch();
    }

    int[] resultados = ps.executeBatch();
    conn.commit();

} catch (SQLException e) {
    e.printStackTrace();
}
```
---

## 9.5 Consultas

### Tipos de ejecución

| Método | Propósito | Retorna |
|---|---|---|
| `executeQuery()` | SELECT | `ResultSet` |
| `executeUpdate()` | INSERT, UPDATE, DELETE, DDL | `int` (filas afectadas) |
| `execute()` | Cualquier SQL (dinámico) | `boolean` (true si hay ResultSet) |

```java
// SELECT
ResultSet rs = ps.executeQuery();

// INSERT, UPDATE, DELETE
int filasAfectadas = ps.executeUpdate();
System.out.println("Filas afectadas: " + filasAfectadas);

// SQL cuyo tipo desconocemos de antemano
boolean esConsulta = stmt.execute(sql);
if (esConsulta) {
    ResultSet rs = stmt.getResultSet();
} else {
    int filas = stmt.getUpdateCount();
}
```

### Procesar ResultSet

Un `ResultSet` es un cursor que inicialmente apunta **antes** de la primera fila. Se avanza con `next()`:

```java
ResultSet rs = ps.executeQuery();

while (rs.next()) {
    long id = rs.getLong("id");           // por nombre de columna
    String nombre = rs.getString("nombre");
    String email = rs.getString(3);       // por índice de columna (1-based)

    System.out.printf("ID: %d, Nombre: %s, Email: %s%n", id, nombre, email);
}
```

### Getters por tipo de columna

| Método | Tipo Java | Tipo SQL recomendado |
|---|---|---|
| `getString()` | `String` | VARCHAR, CHAR, TEXT |
| `getInt()` | `int` | INTEGER |
| `getLong()` | `long` | BIGINT |
| `getDouble()` | `double` | DOUBLE, FLOAT |
| `getBigDecimal()` | `BigDecimal` | DECIMAL, NUMERIC |
| `getBoolean()` | `boolean` | BOOLEAN, BIT |
| `getDate()` | `java.sql.Date` | DATE |
| `getTime()` | `java.sql.Time` | TIME |
| `getTimestamp()` | `java.sql.Timestamp` | TIMESTAMP, DATETIME |
| `getObject()` | `Object` | Cualquier tipo |
| `getBlob()` | `Blob` | BLOB |
| `getClob()` | `Clob` | CLOB |
| `getBytes()` | `byte[]` | BINARY, VARBINARY |

### Manejo de valores NULL

`getInt()` devuelve 0 si la columna es NULL, lo cual es ambiguo. Para distinguir un 0 real de un NULL, usar `wasNull()`:

```java
while (rs.next()) {
    int edad = rs.getInt("edad");
    if (rs.wasNull()) {
        System.out.println("Edad no especificada");
    } else {
        System.out.println("Edad: " + edad);
    }
}
```

Alternativamente, usar `getObject()` que devuelve `null` de Java cuando el valor SQL es NULL:

```java
Integer edad = rs.getObject("edad", Integer.class); // null si la columna es NULL
```

### Scrollable y Updatable ResultSet

Por defecto, un `ResultSet` es `TYPE_FORWARD_ONLY`. Puedes crear ResultSets más potentes:

```java
// Scrollable, insensible a cambios externos, solo lectura
Statement stmt = conn.createStatement(
    ResultSet.TYPE_SCROLL_INSENSITIVE,
    ResultSet.CONCUR_READ_ONLY
);

// Scrollable, insensible a cambios externos, actualizable
Statement stmt = conn.createStatement(
    ResultSet.TYPE_SCROLL_INSENSITIVE,
    ResultSet.CONCUR_UPDATABLE
);
```

| Tipo de Scroll | Descripción |
|---|---|
| `TYPE_FORWARD_ONLY` | Solo `next()`. Más rápido y ligero. |
| `TYPE_SCROLL_INSENSITIVE` | Scroll libre, no ve cambios de otras transacciones. |
| `TYPE_SCROLL_SENSITIVE` | Scroll libre, ve cambios de otras transacciones. |

| Concurrencia | Descripción |
|---|---|
| `CONCUR_READ_ONLY` | Solo lectura. |
| `CONCUR_UPDATABLE` | Permite modificar la fila actual con `updateString()`, `updateRow()`, etc. |

### Recuperar claves auto-generadas

Al insertar con una columna `AUTO_INCREMENT`, puedes recuperar las claves generadas:

```java
String sql = "INSERT INTO usuarios (nombre, email) VALUES (?, ?)";

try (PreparedStatement ps = conn.prepareStatement(sql, Statement.RETURN_GENERATED_KEYS)) {
    ps.setString(1, "María");
    ps.setString(2, "maria@example.com");
    ps.executeUpdate();

    ResultSet generatedKeys = ps.getGeneratedKeys();
    if (generatedKeys.next()) {
        long idGenerado = generatedKeys.getLong(1);
        System.out.println("ID generado: " + idGenerado);
    }
}
```

También puedes especificar qué columnas auto-generadas recuperar:

```java
String[] columnas = {"id", "creado_en"};
PreparedStatement ps = conn.prepareStatement(sql, columnas);
```

---

## 9.6 Transacciones y niveles de aislamiento

### Propiedades ACID — Con ejemplos reales

Una transacción es una unidad lógica de trabajo que debe cumplir ACID:

- **A**tomicidad: todas las operaciones se ejecutan, o ninguna.
  - *Ejemplo:* Transferencia bancaria. Si se descuenta $500 de la cuenta A pero falla al acreditar en B, el descuento debe deshacerse. No puede quedar a medias.

- **C**onsistencia: la base de datos pasa de un estado válido a otro.
  - *Ejemplo:* La suma de todos los saldos antes y después de una transferencia debe ser igual. Las reglas de integridad (FK, CHECK, UNIQUE) se respetan.

- **I**slamiento: las transacciones concurrentes no interfieren entre sí.
  - *Ejemplo:* Dos usuarios compran el último artículo en stock. El aislamiento garantiza que solo uno lo consigue.

- **D**urabilidad: los cambios confirmados persisten incluso ante fallos del sistema.
  - *Ejemplo:* Después de `COMMIT`, si el servidor se apaga, los datos no se pierden. El WAL (Write-Ahead Log) lo garantiza.

### Control de transacciones en JDBC

Por defecto, cada sentencia SQL se confirma automáticamente (**auto-commit = true**). Para agrupar operaciones en una transacción:

```java
Connection conn = DriverManager.getConnection(url, user, pass);
conn.setAutoCommit(false);       // desactivar auto-commit

try {
    // Operación 1
    PreparedStatement ps1 = conn.prepareStatement(
        "UPDATE cuentas SET saldo = saldo - ? WHERE id = ?");
    ps1.setBigDecimal(1, new BigDecimal("500.00"));
    ps1.setLong(2, 1L);
    ps1.executeUpdate();

    // Operación 2
    PreparedStatement ps2 = conn.prepareStatement(
        "UPDATE cuentas SET saldo = saldo + ? WHERE id = ?");
    ps2.setBigDecimal(1, new BigDecimal("500.00"));
    ps2.setLong(2, 2L);
    ps2.executeUpdate();

    conn.commit();                // confirmar ambas operaciones
    System.out.println("Transferencia realizada con éxito");

} catch (SQLException e) {
    conn.rollback();              // deshacer si algo falla
    System.err.println("Error en transferencia, se realizó rollback");
    e.printStackTrace();
} finally {
    conn.setAutoCommit(true);     // restaurar auto-commit
    conn.close();
}
```

### Niveles de aislamiento — Con ejemplos de código

El nivel de aislamiento controla qué anomalías de concurrencia son posibles. A mayor aislamiento, menor concurrencia.

**Las tres anomalías clásicas de concurrencia:**

#### 1. Dirty Read (Lectura sucia)

Una transacción lee datos **no confirmados** de otra. Si la otra hace rollback, la primera leyó datos que nunca existieron.

```java
// --- DEMOSTRACIÓN DE DIRTY READ ---
// Transacción A (READ_UNCOMMITTED)
new Thread(() -> {
    Connection conn = ds.getConnection();
    conn.setAutoCommit(false);
    conn.setTransactionIsolation(Connection.TRANSACTION_READ_UNCOMMITTED);

    // Transacción B ya ha hecho UPDATE pero no COMMIT
    PreparedStatement ps = conn.prepareStatement(
        "SELECT saldo FROM cuentas WHERE id = 1");
    ResultSet rs = ps.executeQuery();
    rs.next();
    BigDecimal saldo = rs.getBigDecimal("saldo");
    // saldo = 5000.00 (pero B aún no ha confirmado...)
    // Si B hace ROLLBACK → este dato nunca existió

    conn.commit();
}).start();

// Transacción B
new Thread(() -> {
    Connection conn = ds.getConnection();
    conn.setAutoCommit(false);

    PreparedStatement ps = conn.prepareStatement(
        "UPDATE cuentas SET saldo = 5000.00 WHERE id = 1");
    ps.executeUpdate();
    // ... algo falla ...
    conn.rollback(); // El saldo real sigue siendo el original (ej. 1000.00)
    conn.close();
}).start();
```

#### 2. Non-Repeatable Read (Lectura no repetible)

Una transacción lee la misma fila **dos veces** y obtiene valores diferentes porque otra la modificó entre ambas lecturas.

```java
// --- DEMOSTRACIÓN DE NON-REPEATABLE READ ---
// Transacción A (READ_COMMITTED)
new Thread(() -> {
    Connection conn = ds.getConnection();
    conn.setAutoCommit(false);
    conn.setTransactionIsolation(Connection.TRANSACTION_READ_COMMITTED);

    // Primera lectura
    PreparedStatement ps = conn.prepareStatement(
        "SELECT saldo FROM cuentas WHERE id = 1");
    ResultSet rs = ps.executeQuery();
    rs.next();
    BigDecimal saldo1 = rs.getBigDecimal("saldo"); // 1000.00

    Thread.sleep(100); // Simular procesamiento

    // Segunda lectura: MISMA fila, MISMA transacción
    ps = conn.prepareStatement(
        "SELECT saldo FROM cuentas WHERE id = 1");
    rs = ps.executeQuery();
    rs.next();
    BigDecimal saldo2 = rs.getBigDecimal("saldo"); // 1500.00 ← ¡CAMBIÓ!

    // Dentro de la misma transacción: saldo1 != saldo2
    // ¡Non-repeatable read!

    conn.commit();
    conn.close();
}).start();

// Transacción B: modifica entre las dos lecturas de A
new Thread(() -> {
    Connection conn = ds.getConnection();
    conn.setAutoCommit(false);

    Thread.sleep(50); // Esperar a que A haga su primera lectura

    PreparedStatement ps = conn.prepareStatement(
        "UPDATE cuentas SET saldo = 1500.00 WHERE id = 1");
    ps.executeUpdate();
    conn.commit(); // Confirma entre las lecturas de A

    conn.close();
}).start();
```

#### 3. Phantom Read (Lectura fantasma)

Una transacción ejecuta la misma consulta **dos veces** y obtiene **diferente número de filas** porque otra transacción insertó o eliminó filas que coinciden con el WHERE.

```java
// --- DEMOSTRACIÓN DE PHANTOM READ ---
// Transacción A (REPEATABLE_READ no bloquea phantoms)
new Thread(() -> {
    Connection conn = ds.getConnection();
    conn.setAutoCommit(false);
    conn.setTransactionIsolation(Connection.TRANSACTION_REPEATABLE_READ);

    // Primera consulta: cuántos usuarios activos
    PreparedStatement ps = conn.prepareStatement(
        "SELECT COUNT(*) FROM usuarios WHERE activo = true");
    ResultSet rs = ps.executeQuery();
    rs.next();
    int count1 = rs.getInt(1); // 10 usuarios

    Thread.sleep(100);

    // Segunda consulta: mismo WHERE
    ps = conn.prepareStatement(
        "SELECT COUNT(*) FROM usuarios WHERE activo = true");
    rs = ps.executeQuery();
    rs.next();
    int count2 = rs.getInt(1); // REPEATABLE_READ: 10 (sin phantom en MySQL/InnoDB)
                                // READ_COMMITTED: 11 (con phantom)

    conn.commit();
    conn.close();
}).start();

// Transacción B: inserta nuevo usuario activo
new Thread(() -> {
    Connection conn = ds.getConnection();
    conn.setAutoCommit(false);

    Thread.sleep(50);

    PreparedStatement ps = conn.prepareStatement(
        "INSERT INTO usuarios (nombre, email, activo) VALUES (?, ?, ?)");
    ps.setString(1, "Nuevo Phantom");
    ps.setString(2, "phantom@test.com");
    ps.setBoolean(3, true);
    ps.executeUpdate();
    conn.commit();

    conn.close();
}).start();
```

### Los 4 niveles de aislamiento

| Constante | Nivel | Dirty Read | Non-repeatable Read | Phantom Read | Rendimiento |
|---|---|---|---|---|---|
| `TRANSACTION_READ_UNCOMMITTED` | Lectura no confirmada | Sí | Sí | Sí | Máximo |
| `TRANSACTION_READ_COMMITTED` | Lectura confirmada | No | Sí | Sí | Alto (default PostgreSQL) |
| `TRANSACTION_REPEATABLE_READ` | Lectura repetible | No | No | Sí (*) | Medio (default MySQL) |
| `TRANSACTION_SERIALIZABLE` | Serializable | No | No | No | Mínimo |

(*) MySQL/MariaDB InnoDB evita phantom reads mediante gap locks incluso en REPEATABLE_READ. PostgreSQL REPEATABLE_READ también evita phantoms. En teoría estándar SQL, REPEATABLE_READ permite phantoms.

**Cómo establecer el nivel de aislamiento en JDBC:**

```java
Connection conn = ds.getConnection();
conn.setAutoCommit(false);

// Nivel por defecto del motor
System.out.println("Nivel actual: " + conn.getTransactionIsolation());

// Cambiar a SERIALIZABLE (máximo aislamiento)
conn.setTransactionIsolation(Connection.TRANSACTION_SERIALIZABLE);
```

### Ejemplo práctico: Comparativa de los 4 niveles con 2 hilos

```java
import java.sql.*;
import javax.sql.DataSource;

/**
 * Demuestra los 4 niveles de aislamiento con 2 hilos concurrentes.
 * Escenario: 2 hilos leen y modifican la misma fila simultáneamente.
 */
public class DemostracionAislamiento {

    private final DataSource dataSource;

    public DemostracionAislamiento(DataSource dataSource) {
        this.dataSource = dataSource;
    }

    public void demostrarNivel(int nivel, String nombreNivel) throws Exception {
        System.out.println("\n=== " + nombreNivel + " ===");

        // Preparar: valor inicial
        try (Connection conn = dataSource.getConnection();
             PreparedStatement ps = conn.prepareStatement(
                 "UPDATE cuentas SET saldo = 100.00 WHERE id = 1")) {
            ps.executeUpdate();
        }

        final BigDecimal[] lecturaHilo1 = new BigDecimal[2];
        final BigDecimal[] lecturaHilo2 = new BigDecimal[2];
        final Object lock = new Object();

        // Hilo 1: lector y modificador (transacción larga)
        Thread t1 = new Thread(() -> {
            try (Connection conn = dataSource.getConnection()) {
                conn.setAutoCommit(false);
                conn.setTransactionIsolation(nivel);

                // Lectura 1
                try (PreparedStatement ps = conn.prepareStatement(
                         "SELECT saldo FROM cuentas WHERE id = 1")) {
                    ResultSet rs = ps.executeQuery();
                    rs.next();
                    lecturaHilo1[0] = rs.getBigDecimal("saldo");
                }

                synchronized (lock) {
                    lock.notify();
                    lock.wait(1000);
                }

                // Lectura 2: ¿mismo valor?
                try (PreparedStatement ps = conn.prepareStatement(
                         "SELECT saldo FROM cuentas WHERE id = 1")) {
                    ResultSet rs = ps.executeQuery();
                    rs.next();
                    lecturaHilo1[1] = rs.getBigDecimal("saldo");
                }

                conn.commit();
            } catch (Exception e) {
                System.err.println("Hilo 1 ERROR: " + e.getMessage());
            }
        });

        // Hilo 2: modificador rápido
        Thread t2 = new Thread(() -> {
            try (Connection conn = dataSource.getConnection()) {
                conn.setAutoCommit(false);
                conn.setTransactionIsolation(nivel);

                synchronized (lock) {
                    lock.wait(1000);
                }

                try (PreparedStatement ps = conn.prepareStatement(
                         "UPDATE cuentas SET saldo = 999.99 WHERE id = 1")) {
                    ps.executeUpdate();
                    lecturaHilo2[0] = new BigDecimal("999.99");
                }

                try (PreparedStatement ps = conn.prepareStatement(
                         "SELECT saldo FROM cuentas WHERE id = 1")) {
                    ResultSet rs = ps.executeQuery();
                    rs.next();
                    lecturaHilo2[1] = rs.getBigDecimal("saldo");
                }

                conn.commit();

                synchronized (lock) {
                    lock.notify();
                }
            } catch (Exception e) {
                System.err.println("Hilo 2 ERROR: " + e.getMessage());
            }
        });

        synchronized (lock) {
            t1.start();
            Thread.sleep(50);
            t2.start();
            lock.wait(5000);
        }

        t1.join(5000);
        t2.join(5000);

        System.out.println("  Hilo 1 → Lectura 1: " + lecturaHilo1[0] +
                           " | Lectura 2: " + lecturaHilo1[1] +
                           " | ¿Cambió? " +
                           (lecturaHilo1[0] != null && !lecturaHilo1[0].equals(lecturaHilo1[1])));
        System.out.println("  Hilo 2 → Escribió: " + lecturaHilo2[0] +
                           " | Leyó: " + lecturaHilo2[1]);
    }

    public static void main(String[] args) throws Exception {
        HikariConfig config = new HikariConfig();
        config.setJdbcUrl("jdbc:h2:mem:test_aislamiento;LOCK_TIMEOUT=10000");
        config.setUsername("sa");
        config.setPassword("");

        try (HikariDataSource ds = new HikariDataSource(config);
             Connection conn = ds.getConnection();
             Statement stmt = conn.createStatement()) {
            stmt.execute("CREATE TABLE cuentas (id BIGINT PRIMARY KEY, saldo DECIMAL(10,2))");
            stmt.execute("INSERT INTO cuentas (id, saldo) VALUES (1, 100.00)");
        }

        HikariDataSource ds = new HikariDataSource(config);
        DemostracionAislamiento demo = new DemostracionAislamiento(ds);

        demo.demostrarNivel(Connection.TRANSACTION_READ_UNCOMMITTED,  "READ_UNCOMMITTED");
        demo.demostrarNivel(Connection.TRANSACTION_READ_COMMITTED,    "READ_COMMITTED");
        demo.demostrarNivel(Connection.TRANSACTION_REPEATABLE_READ,   "REPEATABLE_READ");
        demo.demostrarNivel(Connection.TRANSACTION_SERIALIZABLE,      "SERIALIZABLE");

        ds.close();
    }
}
```

### Propagación de transacciones (concepto de Spring @Transactional)

Cuando múltiples capas y métodos pueden necesitar transacciones, la **propagación** define cómo se comporta una transacción al invocarse desde otra. Aunque es un concepto de Spring (`@Transactional(propagation = ...)`), es crucial entenderlo.

| Propagación | Comportamiento |
|---|---|
| **REQUIRED** (default) | Usa la transacción existente. Si no hay, crea una nueva. La opción más común. |
| **REQUIRES_NEW** | Siempre crea una nueva transacción, suspendiendo la existente. Útil para auditoría: el log debe guardarse aunque el negocio falle. |
| **NESTED** | Crea un savepoint dentro de la transacción existente. Si falla, vuelve al savepoint sin deshacer la transacción entera. |
| **SUPPORTS** | Usa la transacción existente si hay. Si no, ejecuta sin transacción. |
| **NOT_SUPPORTED** | Suspende la transacción existente y ejecuta sin transacción. Útil para operaciones no transaccionales (ej. enviar emails). |
| **NEVER** | Lanza excepción si hay una transacción activa. Garantiza ejecución no transaccional. |
| **MANDATORY** | Lanza excepción si NO hay transacción activa. Obliga al llamante a iniciar la transacción. |

#### Ejemplo conceptual de propagación

```java
// Servicio de negocio: método público → transacción REQUIRED
@Transactional  // (propagation = REQUIRED por defecto)
public void procesarPedido(Pedido pedido) {
    // 1. Guardar pedido (REQUIRED → misma transacción)
    pedidoRepo.save(pedido);

    // 2. Guardar log de auditoría (REQUIRES_NEW → NUEVA transacción independiente)
    auditoriaService.registrar(pedido);
    // Si esto falla, el pedido YA ESTÁ guardado (transacciones independientes)

    // 3. Actualizar inventario (NESTED: si falla, vuelve al savepoint)
    try {
        inventarioService.descontar(pedido);
    } catch (Exception e) {
        // Rollback parcial: solo se deshace el descuento, el pedido sigue guardado
    }
}
```

#### Implementación manual en JDBC (equivalente a REQUIRES_NEW)

```java
public void procesarPedido(Pedido pedido, DataSource ds) throws SQLException {
    // Transacción principal
    Connection connPrincipal = ds.getConnection();
    connPrincipal.setAutoCommit(false);

    try {
        guardarPedido(connPrincipal, pedido);

        // Abrir SEGUNDA transacción independiente para auditoría
        Connection connAuditoria = ds.getConnection();
        connAuditoria.setAutoCommit(false);
        try {
            guardarAuditoria(connAuditoria, pedido);
            connAuditoria.commit(); // Commit independiente
        } catch (Exception e) {
            connAuditoria.rollback();
            // No hacer rollback de la principal — auditoría falló, pedido sigue
        } finally {
            connAuditoria.close();
        }

        connPrincipal.commit();
    } catch (Exception e) {
        connPrincipal.rollback();
        throw e;
    } finally {
        connPrincipal.close();
    }
}
```

### Savepoints: rollback parcial dentro de una transacción

Un *savepoint* marca un punto intermedio al que se puede volver sin deshacer toda la transacción:

```java
conn.setAutoCommit(false);

Savepoint sp1 = conn.setSavepoint("inicio");
Savepoint sp2 = conn.setSavepoint("despues_validacion");

try {
    // Paso 1: validar datos
    // ...

    // Paso 2: insertar registro principal
    PreparedStatement ps = conn.prepareStatement(
        "INSERT INTO pedidos (descripcion, total) VALUES (?, ?)");
    ps.setString(1, "Pedido 123");
    ps.setBigDecimal(2, new BigDecimal("500.00"));
    ps.executeUpdate();

    // Paso 3: insertar items (puede fallar)
    try {
        for (ItemPedido item : items) {
            insertarItem(conn, item);
        }
    } catch (SQLException e) {
        System.err.println("Error insertando items: " + e.getMessage());
        // Volver al savepoint sp2: el pedido principal sobrevive
        conn.rollback(sp2);

        // Reintentar con otra estrategia
        insertarItemsEnLote(conn, items);
    }

    // Si todo fue bien
    conn.releaseSavepoint(sp1);
    conn.releaseSavepoint(sp2);
    conn.commit();

} catch (SQLException e) {
    conn.rollback(); // rollback total si falla algo no cubierto
    e.printStackTrace();
}
```

**Limitaciones de savepoints:**
- Consumen recursos en la base de datos.
- No todos los drivers los soportan completamente.
- Spring `NESTED` usa savepoints internamente.

### Deadlocks y estrategia de retry

Un *deadlock* ocurre cuando dos transacciones se bloquean mutuamente esperando locks que la otra posee:

```
Transacción A: UPDATE cuentas SET saldo = saldo - 100 WHERE id = 1     (bloquea id=1)
                UPDATE cuentas SET saldo = saldo + 100 WHERE id = 2    (espera id=2, bloqueado por B)

Transacción B: UPDATE cuentas SET saldo = saldo - 50 WHERE id = 2      (bloquea id=2)
                UPDATE cuentas SET saldo = saldo + 50 WHERE id = 1     (espera id=1, bloqueado por A)

→ DEADLOCK: ninguna puede continuar. La BD aborta una de ellas.
```

**La aplicación debe reintentar.**

```java
/**
 * Ejecuta una operación transaccional con reintentos en caso de deadlock.
 */
public class TransaccionConRetry {

    private static final int MAX_RETRIES = 3;
    private static final long RETRY_DELAY_MS = 200;
    private final DataSource dataSource;

    public TransaccionConRetry(DataSource dataSource) {
        this.dataSource = dataSource;
    }

    @FunctionalInterface
    public interface OperacionTransaccional<T> {
        T ejecutar(Connection conn) throws SQLException;
    }

    public <T> T ejecutarConRetry(OperacionTransaccional<T> operacion) throws SQLException {
        int intento = 0;

        while (true) {
            try (Connection conn = dataSource.getConnection()) {
                conn.setAutoCommit(false);
                conn.setTransactionIsolation(Connection.TRANSACTION_READ_COMMITTED);

                try {
                    T resultado = operacion.ejecutar(conn);
                    conn.commit();
                    return resultado;

                } catch (SQLException e) {
                    try { conn.rollback(); } catch (SQLException ignored) {}

                    if (esDeadlock(e) && intento < MAX_RETRIES) {
                        intento++;
                        long delay = RETRY_DELAY_MS * (long) Math.pow(2, intento); // backoff exponencial
                        System.out.println("Deadlock detectado, reintento " + intento +
                                         " de " + MAX_RETRIES + " en " + delay + "ms");

                        try { Thread.sleep(delay); } catch (InterruptedException ie) {
                            Thread.currentThread().interrupt();
                            throw new SQLException("Interrumpido durante retry", ie);
                        }
                        continue;
                    }
                    throw e;
                }
            }
        }
    }

    private boolean esDeadlock(SQLException e) {
        String sqlState = e.getSQLState();
        return "40001".equals(sqlState) || "40P01".equals(sqlState) ||
               (e.getErrorCode() == 1213) || (e.getErrorCode() == 1205);
    }
}
```

**Uso:**

```java
TransaccionConRetry ejecutor = new TransaccionConRetry(dataSource);

ejecutor.ejecutarConRetry(conn -> {
    PreparedStatement ps1 = conn.prepareStatement(
        "UPDATE cuentas SET saldo = saldo - ? WHERE id = ?");
    ps1.setBigDecimal(1, monto);
    ps1.setLong(2, cuentaOrigen);
    ps1.executeUpdate();

    PreparedStatement ps2 = conn.prepareStatement(
        "UPDATE cuentas SET saldo = saldo + ? WHERE id = ?");
    ps2.setBigDecimal(1, monto);
    ps2.setLong(2, cuentaDestino);
    ps2.executeUpdate();

    return null;
});
```

### Timeout de consultas

Para evitar que consultas lentas bloqueen recursos indefinidamente:

```java
// Timeout a nivel de Statement (segundos)
ps.setQueryTimeout(30); // lanza SQLTimeoutException si excede 30s

// Timeout a nivel de transacción (motor específico)
// PostgreSQL:
stmt.execute("SET statement_timeout = '30s'");
stmt.execute("SET lock_timeout = '10s'");

// MySQL:
stmt.execute("SET innodb_lock_wait_timeout = 5");
stmt.execute("SET max_execution_time = 30000");
```

---

## 9.7 CallableStatement

`CallableStatement` invoca **procedimientos almacenados** y **funciones** de la base de datos.

### Sintaxis de invocación

```java
// Procedimiento sin parámetros
CallableStatement cs = conn.prepareCall("{call nombre_procedimiento()}");

// Procedimiento con parámetros
CallableStatement cs = conn.prepareCall("{call nombre_procedimiento(?, ?, ?)}");

// Función que retorna valor
CallableStatement cs = conn.prepareCall("{? = call nombre_funcion(?)}");
```

### Parámetros IN, OUT e INOUT

Procedimiento almacenado en MySQL:

```sql
DELIMITER $$
CREATE PROCEDURE obtener_datos_usuario(
    IN  p_id     BIGINT,
    OUT p_nombre VARCHAR(100),
    OUT p_email  VARCHAR(100)
)
BEGIN
    SELECT nombre, email INTO p_nombre, p_email
    FROM usuarios WHERE id = p_id;
END$$
DELIMITER ;
```

Invocación desde Java:

```java
String sql = "{call obtener_datos_usuario(?, ?, ?)}";

try (Connection conn = DriverManager.getConnection(url, user, pass);
     CallableStatement cs = conn.prepareCall(sql)) {

    cs.setLong(1, 42L);                         // IN
    cs.registerOutParameter(2, Types.VARCHAR);   // OUT: p_nombre
    cs.registerOutParameter(3, Types.VARCHAR);   // OUT: p_email
    cs.execute();

    String nombre = cs.getString(2);
    String email = cs.getString(3);
    System.out.printf("Nombre: %s, Email: %s%n", nombre, email);
}
```

### Parámetro INOUT

```sql
CREATE PROCEDURE duplicar_saldo(INOUT saldo DECIMAL(10,2))
BEGIN
    SET saldo = saldo * 2;
END
```

```java
cs.setBigDecimal(1, new BigDecimal("100.00"));
cs.registerOutParameter(1, Types.DECIMAL);
cs.execute();
BigDecimal nuevoSaldo = cs.getBigDecimal(1); // 200.00
```

### Múltiples ResultSets desde un procedimiento

Un procedimiento puede devolver varios `ResultSet`:

```java
boolean hayResultados = cs.execute();

while (true) {
    if (hayResultados) {
        ResultSet rs = cs.getResultSet();
        // procesar rs...
        rs.close();
    } else {
        if (cs.getUpdateCount() == -1) break;
    }
    hayResultados = cs.getMoreResults();
}
```
---

## 9.8 Metadatos

### DatabaseMetaData

Proporciona información sobre la base de datos: tablas, columnas, procedimientos, capacidades del driver:

```java
try (Connection conn = DriverManager.getConnection(url, user, pass)) {

    DatabaseMetaData meta = conn.getMetaData();

    // Información del driver y la BD
    System.out.println("Driver: " + meta.getDriverName() + " " + meta.getDriverVersion());
    System.out.println("BD: " + meta.getDatabaseProductName()
        + " " + meta.getDatabaseProductVersion());
    System.out.println("Soporta transacciones: " + meta.supportsTransactions());
    System.out.println("Soporta batch updates: " + meta.supportsBatchUpdates());
    System.out.println("Soporta savepoints: " + meta.supportsSavepoints());
    System.out.println("Nivel de aislamiento default: " + meta.getDefaultTransactionIsolation());

    // Listar tablas
    ResultSet tablas = meta.getTables(null, null, "%", new String[]{"TABLE"});
    while (tablas.next()) {
        System.out.println("Tabla: " + tablas.getString("TABLE_NAME"));
    }

    // Listar columnas de una tabla
    ResultSet columnas = meta.getColumns(null, null, "usuarios", "%");
    while (columnas.next()) {
        System.out.printf("Columna: %s, Tipo: %s, Nullable: %s%n",
            columnas.getString("COLUMN_NAME"),
            columnas.getString("TYPE_NAME"),
            columnas.getInt("NULLABLE"));
    }

    // Procedimientos almacenados
    ResultSet procs = meta.getProcedures(null, null, "%");
    while (procs.next()) {
        System.out.println("Procedimiento: " + procs.getString("PROCEDURE_NAME"));
    }

    // Claves primarias
    ResultSet pks = meta.getPrimaryKeys(null, null, "usuarios");
    while (pks.next()) {
        System.out.printf("PK: %s (col: %s, seq: %d)%n",
            pks.getString("PK_NAME"),
            pks.getString("COLUMN_NAME"),
            pks.getShort("KEY_SEQ"));
    }

    // Claves foráneas (imported keys)
    ResultSet fks = meta.getImportedKeys(null, null, "pedidos");
    while (fks.next()) {
        System.out.printf("FK: %s → %s.%s%n",
            fks.getString("FKCOLUMN_NAME"),
            fks.getString("PKTABLE_NAME"),
            fks.getString("PKCOLUMN_NAME"));
    }

    // Índices
    ResultSet indices = meta.getIndexInfo(null, null, "usuarios", false, false);
    while (indices.next()) {
        System.out.printf("Índice: %s, columna: %s%n",
            indices.getString("INDEX_NAME"),
            indices.getString("COLUMN_NAME"));
    }
}
```

### ResultSetMetaData

Proporciona información sobre la estructura de un `ResultSet`:

```java
ResultSet rs = ps.executeQuery();
ResultSetMetaData rsmd = rs.getMetaData();

int columnCount = rsmd.getColumnCount();

for (int i = 1; i <= columnCount; i++) {
    System.out.printf("Columna %d: %s (%s), precisión: %d, escala: %d, nullable: %d%n",
        i,
        rsmd.getColumnName(i),      // nombre
        rsmd.getColumnTypeName(i),  // tipo SQL (ej: VARCHAR)
        rsmd.getPrecision(i),       // precisión
        rsmd.getScale(i),           // escala
        rsmd.isNullable(i));        // nullable
}

// Genérico: convertir ResultSet a lista de mapas dinámicamente
List<Map<String, Object>> filas = new ArrayList<>();
while (rs.next()) {
    Map<String, Object> fila = new HashMap<>();
    for (int i = 1; i <= columnCount; i++) {
        fila.put(rsmd.getColumnName(i), rs.getObject(i));
    }
    filas.add(fila);
}
```

---

## 9.9 RowSet

`RowSet` extiende `ResultSet` añadiendo capacidades que `ResultSet` no tiene: es scrollable, actualizable y puede operar desconectado.

### Tipos de RowSet

#### JdbcRowSet (conectado)

Mantiene la conexión abierta. Ideal para UIs o navegación bidireccional con actualización directa:

```java
JdbcRowSet rowSet = RowSetProvider.newFactory().createJdbcRowSet();
rowSet.setUrl("jdbc:mysql://localhost:3306/mi_base");
rowSet.setUsername("admin");
rowSet.setPassword("secreto");
rowSet.setCommand("SELECT id, nombre, email FROM usuarios");
rowSet.execute();

// Navegación libre
while (rowSet.next()) {
    if (rowSet.getInt("id") == 5) {
        rowSet.updateString("nombre", "Nuevo Nombre");
        rowSet.updateRow();
    }
}
rowSet.absolute(3);
rowSet.previous();
rowSet.first();
rowSet.last();
rowSet.close();
```

#### CachedRowSet (desconectado)

Carga los datos en memoria y cierra la conexión. Es `Serializable`, ideal para transferir datos entre capas:

```java
CachedRowSet cached = RowSetProvider.newFactory().createCachedRowSet();
cached.setUrl("jdbc:mysql://localhost:3306/mi_base");
cached.setUsername("admin");
cached.setPassword("secreto");
cached.setCommand("SELECT * FROM usuarios");
cached.execute(); // carga datos y cierra conexión

// Propagar cambios de vuelta a la BD
cached.acceptChanges(conn); // necesita una conexión

// Serialización para enviar por red
try (ObjectOutputStream oos = new ObjectOutputStream(socket.getOutputStream())) {
    oos.writeObject(cached);
}
```

#### FilteredRowSet y JoinRowSet

`FilteredRowSet` permite filtrar filas en memoria sin consultar la BD. `JoinRowSet` combina múltiples RowSets como un SQL JOIN.

### Comparativa RowSet vs ResultSet

| Característica | ResultSet | JdbcRowSet | CachedRowSet |
|---|---|---|---|
| Conexión requerida | Siempre | Siempre | Solo al cargar/guardar |
| Scrollable | No (forward-only) | Sí | Sí |
| Updatable | No | Sí | Sí (offline, luego sync) |
| Serializable | No | No | Sí |
| Adecuado para | Procesar y liberar | UI interactiva | Capa de presentación / web |

---

## 9.10 Patrón DAO (Data Access Object)

### Propósito

El patrón DAO separa la lógica de persistencia de la lógica de negocio. Centraliza todas las operaciones CRUD en una clase dedicada.

### Estructura

```
┌──────────────────┐     usa      ┌──────────────────┐     usa      ┌──────────────────┐
│   Servicio       │ ──────────▶  │       DAO        │ ──────────▶  │     JDBC         │
│ (lógica negocio) │              │ (acceso a datos)  │              │ (SQL, DB)        │
└──────────────────┘              └──────────────────┘              └──────────────────┘
          │                               │
          ▼                               ▼
┌──────────────────┐              ┌──────────────────┐
│       DTO        │              │  DataSource /    │
│ (datos puros)    │              │  Connection Pool │
└──────────────────┘              └──────────────────┘
```

- **DTO** (*Data Transfer Object*): clase simple con atributos, getters, setters, sin lógica. Transporta datos entre capas.
- **DAO**: contiene SQL y lógica de mapeo entre `ResultSet` y DTO.
- **DataSource / Connection Pool**: proporciona conexiones.

### Implementación completa: UsuarioDAO

Supongamos la tabla:

```sql
CREATE TABLE usuarios (
    id       BIGINT AUTO_INCREMENT PRIMARY KEY,
    nombre   VARCHAR(100) NOT NULL,
    email    VARCHAR(150) NOT NULL UNIQUE,
    activo   BOOLEAN NOT NULL DEFAULT TRUE,
    creado_en TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

#### Paso 1: DTO

```java
import java.time.LocalDateTime;

public class Usuario {

    private Long id;
    private String nombre;
    private String email;
    private boolean activo;
    private LocalDateTime creadoEn;

    public Usuario() {}

    public Usuario(Long id, String nombre, String email, boolean activo, LocalDateTime creadoEn) {
        this.id = id;
        this.nombre = nombre;
        this.email = email;
        this.activo = activo;
        this.creadoEn = creadoEn;
    }

    public Long getId() { return id; }
    public void setId(Long id) { this.id = id; }
    public String getNombre() { return nombre; }
    public void setNombre(String nombre) { this.nombre = nombre; }
    public String getEmail() { return email; }
    public void setEmail(String email) { this.email = email; }
    public boolean isActivo() { return activo; }
    public void setActivo(boolean activo) { this.activo = activo; }
    public LocalDateTime getCreadoEn() { return creadoEn; }
    public void setCreadoEn(LocalDateTime creadoEn) { this.creadoEn = creadoEn; }

    @Override
    public String toString() {
        return String.format("Usuario[id=%d, nombre=%s, email=%s, activo=%s, creadoEn=%s]",
                id, nombre, email, activo, creadoEn);
    }
}
```

#### Paso 2: Interfaces

```java
import java.util.List;
import java.util.Optional;

public interface CrudRepository<T, ID> {
    T save(T entity);
    Optional<T> findById(ID id);
    List<T> findAll();
    void update(T entity);
    void deleteById(ID id);
}
```

```java
import java.util.List;

public interface UsuarioRepository extends CrudRepository<Usuario, Long> {
    Optional<Usuario> findByEmail(String email);
    List<Usuario> findByActivo(boolean activo);
    List<Usuario> searchByNombre(String keyword);
    long count();
}
```

#### Paso 3: Implementación concreta del DAO

```java
import java.sql.*;
import java.time.LocalDateTime;
import java.util.ArrayList;
import java.util.List;
import java.util.Optional;
import javax.sql.DataSource;

/**
 * Implementación JDBC del repositorio de usuarios.
 */
public class UsuarioDAO implements UsuarioRepository {

    private final DataSource dataSource;

    public UsuarioDAO(DataSource dataSource) {
        this.dataSource = dataSource;
    }

    @Override
    public Usuario save(Usuario usuario) {
        String sql = "INSERT INTO usuarios (nombre, email, activo) VALUES (?, ?, ?)";

        try (Connection conn = dataSource.getConnection();
             PreparedStatement ps = conn.prepareStatement(sql, Statement.RETURN_GENERATED_KEYS)) {

            ps.setString(1, usuario.getNombre());
            ps.setString(2, usuario.getEmail());
            ps.setBoolean(3, usuario.isActivo());

            int filas = ps.executeUpdate();
            if (filas == 0) {
                throw new SQLException("No se pudo insertar el usuario");
            }

            ResultSet generatedKeys = ps.getGeneratedKeys();
            if (generatedKeys.next()) {
                usuario.setId(generatedKeys.getLong(1));
            }

            return usuario;

        } catch (SQLException e) {
            throw new RuntimeException("Error al guardar usuario", e);
        }
    }

    @Override
    public Optional<Usuario> findById(Long id) {
        String sql = "SELECT id, nombre, email, activo, creado_en FROM usuarios WHERE id = ?";

        try (Connection conn = dataSource.getConnection();
             PreparedStatement ps = conn.prepareStatement(sql)) {

            ps.setLong(1, id);
            ResultSet rs = ps.executeQuery();
            if (rs.next()) {
                return Optional.of(mapRow(rs));
            }
            return Optional.empty();

        } catch (SQLException e) {
            throw new RuntimeException("Error al buscar usuario por id=" + id, e);
        }
    }

    @Override
    public List<Usuario> findAll() {
        String sql = "SELECT id, nombre, email, activo, creado_en FROM usuarios ORDER BY id";
        List<Usuario> usuarios = new ArrayList<>();

        try (Connection conn = dataSource.getConnection();
             PreparedStatement ps = conn.prepareStatement(sql);
             ResultSet rs = ps.executeQuery()) {

            while (rs.next()) {
                usuarios.add(mapRow(rs));
            }

        } catch (SQLException e) {
            throw new RuntimeException("Error al listar usuarios", e);
        }

        return usuarios;
    }

    @Override
    public Optional<Usuario> findByEmail(String email) {
        String sql = "SELECT id, nombre, email, activo, creado_en FROM usuarios WHERE email = ?";

        try (Connection conn = dataSource.getConnection();
             PreparedStatement ps = conn.prepareStatement(sql)) {

            ps.setString(1, email);
            ResultSet rs = ps.executeQuery();
            if (rs.next()) {
                return Optional.of(mapRow(rs));
            }
            return Optional.empty();

        } catch (SQLException e) {
            throw new RuntimeException("Error al buscar usuario por email=" + email, e);
        }
    }

    @Override
    public List<Usuario> findByActivo(boolean activo) {
        String sql = "SELECT id, nombre, email, activo, creado_en FROM usuarios WHERE activo = ?";
        List<Usuario> usuarios = new ArrayList<>();

        try (Connection conn = dataSource.getConnection();
             PreparedStatement ps = conn.prepareStatement(sql)) {

            ps.setBoolean(1, activo);
            ResultSet rs = ps.executeQuery();
            while (rs.next()) {
                usuarios.add(mapRow(rs));
            }

        } catch (SQLException e) {
            throw new RuntimeException("Error al buscar usuarios activos=" + activo, e);
        }

        return usuarios;
    }

    @Override
    public List<Usuario> searchByNombre(String keyword) {
        String sql = "SELECT id, nombre, email, activo, creado_en FROM usuarios "
                   + "WHERE nombre LIKE ? ORDER BY nombre";
        List<Usuario> usuarios = new ArrayList<>();

        try (Connection conn = dataSource.getConnection();
             PreparedStatement ps = conn.prepareStatement(sql)) {

            ps.setString(1, "%" + keyword + "%");
            ResultSet rs = ps.executeQuery();
            while (rs.next()) {
                usuarios.add(mapRow(rs));
            }

        } catch (SQLException e) {
            throw new RuntimeException("Error al buscar usuarios por nombre: " + keyword, e);
        }

        return usuarios;
    }

    @Override
    public long count() {
        String sql = "SELECT COUNT(*) FROM usuarios";

        try (Connection conn = dataSource.getConnection();
             PreparedStatement ps = conn.prepareStatement(sql);
             ResultSet rs = ps.executeQuery()) {

            if (rs.next()) {
                return rs.getLong(1);
            }
            return 0;

        } catch (SQLException e) {
            throw new RuntimeException("Error al contar usuarios", e);
        }
    }

    @Override
    public void update(Usuario usuario) {
        String sql = "UPDATE usuarios SET nombre = ?, email = ?, activo = ? WHERE id = ?";

        try (Connection conn = dataSource.getConnection();
             PreparedStatement ps = conn.prepareStatement(sql)) {

            ps.setString(1, usuario.getNombre());
            ps.setString(2, usuario.getEmail());
            ps.setBoolean(3, usuario.isActivo());
            ps.setLong(4, usuario.getId());

            int filas = ps.executeUpdate();
            if (filas == 0) {
                throw new SQLException("No se encontró el usuario con id=" + usuario.getId());
            }

        } catch (SQLException e) {
            throw new RuntimeException("Error al actualizar usuario", e);
        }
    }

    @Override
    public void deleteById(Long id) {
        String sql = "DELETE FROM usuarios WHERE id = ?";

        try (Connection conn = dataSource.getConnection();
             PreparedStatement ps = conn.prepareStatement(sql)) {

            ps.setLong(1, id);
            int filas = ps.executeUpdate();
            if (filas == 0) {
                throw new SQLException("No se encontró el usuario con id=" + id);
            }

        } catch (SQLException e) {
            throw new RuntimeException("Error al eliminar usuario con id=" + id, e);
        }
    }

    /**
     * Convierte la fila actual del ResultSet en un objeto Usuario.
     */
    private Usuario mapRow(ResultSet rs) throws SQLException {
        Usuario usuario = new Usuario();
        usuario.setId(rs.getLong("id"));
        if (rs.wasNull()) {
            throw new SQLException("id no puede ser NULL");
        }
        usuario.setNombre(rs.getString("nombre"));
        usuario.setEmail(rs.getString("email"));
        usuario.setActivo(rs.getBoolean("activo"));

        Timestamp ts = rs.getTimestamp("creado_en");
        if (ts != null) {
            usuario.setCreadoEn(ts.toLocalDateTime());
        }

        return usuario;
    }
}
```

#### Paso 4: Uso del DAO desde una clase de servicio

```java
import com.zaxxer.hikari.HikariConfig;
import com.zaxxer.hikari.HikariDataSource;
import java.sql.Connection;
import java.sql.SQLException;
import java.sql.Statement;
import java.util.List;
import java.util.Optional;

public class UsuarioService {

    private final UsuarioRepository usuarioRepo;

    public UsuarioService(UsuarioRepository usuarioRepo) {
        this.usuarioRepo = usuarioRepo;
    }

    public Usuario registrar(String nombre, String email) {
        if (nombre == null || nombre.isBlank()) {
            throw new IllegalArgumentException("El nombre es obligatorio");
        }

        Optional<Usuario> existente = usuarioRepo.findByEmail(email);
        if (existente.isPresent()) {
            throw new IllegalArgumentException("El email ya está registrado");
        }

        Usuario usuario = new Usuario();
        usuario.setNombre(nombre);
        usuario.setEmail(email);
        usuario.setActivo(true);

        return usuarioRepo.save(usuario);
    }

    public void desactivar(Long id) {
        Optional<Usuario> opt = usuarioRepo.findById(id);
        if (opt.isEmpty()) {
            throw new IllegalArgumentException("Usuario no encontrado: " + id);
        }

        Usuario usuario = opt.get();
        usuario.setActivo(false);
        usuarioRepo.update(usuario);
    }

    public List<Usuario> listarActivos() {
        return usuarioRepo.findByActivo(true);
    }

    public Optional<Usuario> buscarPorId(Long id) {
        return usuarioRepo.findById(id);
    }

    public void eliminar(Long id) {
        usuarioRepo.deleteById(id);
    }

    public static void main(String[] args) {
        HikariConfig config = new HikariConfig();
        config.setJdbcUrl("jdbc:h2:mem:demo;DB_CLOSE_DELAY=-1");
        config.setUsername("sa");
        config.setPassword("");

        HikariDataSource ds = new HikariDataSource(config);

        // Crear tabla
        try (Connection conn = ds.getConnection();
             Statement stmt = conn.createStatement()) {
            stmt.execute("""
                CREATE TABLE usuarios (
                    id       BIGINT AUTO_INCREMENT PRIMARY KEY,
                    nombre   VARCHAR(100) NOT NULL,
                    email    VARCHAR(150) NOT NULL UNIQUE,
                    activo   BOOLEAN NOT NULL DEFAULT TRUE,
                    creado_en TIMESTAMP DEFAULT CURRENT_TIMESTAMP
                )
                """);
        } catch (SQLException e) {
            e.printStackTrace();
        }

        UsuarioRepository repo = new UsuarioDAO(ds);
        UsuarioService service = new UsuarioService(repo);

        Usuario u1 = service.registrar("Andrés", "andres@example.com");
        Usuario u2 = service.registrar("María",  "maria@example.com");
        Usuario u3 = service.registrar("Carlos", "carlos@example.com");
        System.out.println("Usuarios registrados: " + u1.getId() + ", " + u2.getId() + ", " + u3.getId());

        System.out.println("\n--- Todos los usuarios ---");
        service.listarActivos().forEach(System.out::println);

        service.desactivar(u2.getId());
        System.out.println("\n--- Usuario desactivado: id=" + u2.getId() + " ---");

        System.out.println("\n--- Activos ---");
        service.listarActivos().forEach(System.out::println);

        System.out.println("\n--- Búsqueda por ID ---");
        service.buscarPorId(u1.getId()).ifPresent(System.out::println);

        service.eliminar(u3.getId());
        System.out.println("\n--- Después de eliminar id=" + u3.getId() + " ---");
        System.out.println("Total: " + repo.count() + " usuarios");

        ds.close();
    }
}
```
---

## 9.11 Batch Operations y rendimiento

### El problema de las inserciones individuales

Cuando necesitas insertar miles o millones de registros, hacer un `executeUpdate()` por cada fila es extremadamente ineficiente. Cada llamada implica:

1. Enviar el SQL por red (round-trip).
2. Parsear y compilar el SQL en BD.
3. Ejecutar la inserción.
4. Enviar confirmación.

Con 10 000 registros: 10 000 round-trips de red. Con un ping de 1ms: ~10 segundos solo en latencia.

### addBatch() + executeBatch(): La solución

```java
String sql = "INSERT INTO mediciones (sensor_id, valor, timestamp) VALUES (?, ?, ?)";

try (Connection conn = dataSource.getConnection();
     PreparedStatement ps = conn.prepareStatement(sql)) {

    conn.setAutoCommit(false);

    for (int i = 0; i < 10_000; i++) {
        ps.setLong(1, sensorId);
        ps.setDouble(2, Math.random() * 100);
        ps.setTimestamp(3, Timestamp.valueOf(LocalDateTime.now()));
        ps.addBatch();

        // Enviar lote cada 1000 registros para no saturar memoria
        if (i % 1000 == 0 && i > 0) {
            ps.executeBatch();
            conn.commit();
        }
    }

    ps.executeBatch(); // Lote final
    conn.commit();

} catch (SQLException e) {
    e.printStackTrace();
}
```

### Benchmark: Inserciones uno a uno vs Batch

```java
import com.zaxxer.hikari.HikariConfig;
import com.zaxxer.hikari.HikariDataSource;
import java.sql.*;
import javax.sql.DataSource;

/**
 * Comparativa: inserción individual vs batch de 100 vs 500 vs 1000.
 * 
 * Resultados típicos (H2 en memoria):
 *   10,000 inserts individuales:   ~800-1200 ms
 *   10,000 inserts batch 100:      ~50-80 ms
 *   10,000 inserts batch 1000:     ~30-50 ms
 * 
 * Con PostgreSQL/MySQL (latencia de red): el speedup es aún mayor (50x-100x)
 */
public class BatchBenchmark {

    private static final int TOTAL_RECORDS = 10_000;

    public static void main(String[] args) throws Exception {
        HikariConfig config = new HikariConfig();
        config.setJdbcUrl("jdbc:h2:mem:benchmark;DB_CLOSE_DELAY=-1");
        config.setUsername("sa");
        config.setPassword("");
        config.setMaximumPoolSize(5);

        try (HikariDataSource ds = new HikariDataSource(config)) {
            crearTabla(ds);
            warmup(ds);

            benchmarkIndividual(ds);
            benchmarkBatch(ds, 100);
            benchmarkBatch(ds, 500);
            benchmarkBatch(ds, 1000);
        }
    }

    private static void crearTabla(DataSource ds) throws SQLException {
        try (Connection conn = ds.getConnection();
             Statement stmt = conn.createStatement()) {
            stmt.execute("DROP TABLE IF EXISTS mediciones");
            stmt.execute("""
                CREATE TABLE mediciones (
                    id BIGINT AUTO_INCREMENT PRIMARY KEY,
                    sensor_id BIGINT NOT NULL,
                    valor DOUBLE NOT NULL,
                    timestamp TIMESTAMP NOT NULL
                )
                """);
        }
    }

    private static void warmup(DataSource ds) throws SQLException {
        insertarIndividual(ds, 500);
        limpiarTabla(ds);
    }

    private static void limpiarTabla(DataSource ds) throws SQLException {
        try (Connection conn = ds.getConnection();
             Statement stmt = conn.createStatement()) {
            stmt.execute("TRUNCATE TABLE mediciones");
        }
    }

    private static void benchmarkIndividual(DataSource ds) throws SQLException {
        limpiarTabla(ds);
        long inicio = System.currentTimeMillis();
        insertarIndividual(ds, TOTAL_RECORDS);
        long fin = System.currentTimeMillis();

        System.out.printf("Individual (%d inserts): %d ms | %.1f inserts/ms%n",
                TOTAL_RECORDS, (fin - inicio),
                (double) TOTAL_RECORDS / (fin - inicio));
    }

    private static void insertarIndividual(DataSource ds, int total) throws SQLException {
        String sql = "INSERT INTO mediciones (sensor_id, valor, timestamp) VALUES (?, ?, ?)";
        try (Connection conn = ds.getConnection();
             PreparedStatement ps = conn.prepareStatement(sql)) {

            conn.setAutoCommit(false);
            for (int i = 0; i < total; i++) {
                ps.setLong(1, i % 10);
                ps.setDouble(2, Math.random() * 100);
                ps.setTimestamp(3, new Timestamp(System.currentTimeMillis()));
                ps.executeUpdate();
            }
            conn.commit();
        }
    }

    private static void benchmarkBatch(DataSource ds, int batchSize) throws SQLException {
        limpiarTabla(ds);
        long inicio = System.currentTimeMillis();
        insertarBatch(ds, TOTAL_RECORDS, batchSize);
        long fin = System.currentTimeMillis();

        System.out.printf("Batch-%d (%d inserts): %d ms | %.1f inserts/ms%n",
                batchSize, TOTAL_RECORDS, (fin - inicio),
                (double) TOTAL_RECORDS / (fin - inicio));
    }

    private static void insertarBatch(DataSource ds, int total, int batchSize) throws SQLException {
        String sql = "INSERT INTO mediciones (sensor_id, valor, timestamp) VALUES (?, ?, ?)";
        try (Connection conn = ds.getConnection();
             PreparedStatement ps = conn.prepareStatement(sql)) {

            conn.setAutoCommit(false);
            for (int i = 0; i < total; i++) {
                ps.setLong(1, i % 10);
                ps.setDouble(2, Math.random() * 100);
                ps.setTimestamp(3, new Timestamp(System.currentTimeMillis()));
                ps.addBatch();

                if ((i + 1) % batchSize == 0) {
                    ps.executeBatch();
                }
            }
            ps.executeBatch(); // Resto
            conn.commit();
        }
    }
}
```

**Resultados típicos:**

```
Individual (10000 inserts): 892 ms | 11.2 inserts/ms
Batch-100 (10000 inserts):  62 ms  | 161.3 inserts/ms
Batch-500 (10000 inserts):  45 ms  | 222.2 inserts/ms
Batch-1000 (10000 inserts): 38 ms  | 263.2 inserts/ms
```

Con PostgreSQL/MySQL (con latencia de red el speedup puede ser 50x-100x mayor).

### PreparedStatement Caching

Además del batch, el caching del PreparedStatement ahorra la re-compilación del SQL:

```java
// Configurar caching en HikariCP (delegado al driver)
config.addDataSourceProperty("cachePrepStmts", "true");
config.addDataSourceProperty("prepStmtCacheSize", "250");
config.addDataSourceProperty("prepStmtCacheSqlLimit", "2048");
config.addDataSourceProperty("useServerPrepStmts", "true");

// PostgreSQL JDBC:
config.addDataSourceProperty("prepareThreshold", "1");
```

### fetchSize y su impacto en memoria/rendimiento

`fetchSize` controla cuántas filas recupera el driver en cada viaje de red:

```java
// PROBLEMA: sin fetchSize, el driver puede cargar TODAS las filas en memoria
PreparedStatement ps = conn.prepareStatement("SELECT * FROM tabla_grande");
ResultSet rs = ps.executeQuery();
// Si la tabla tiene 2 millones de filas → OutOfMemoryError

// SOLUCIÓN: fetchSize controla el streaming
ps.setFetchSize(1000);
ResultSet rs = ps.executeQuery();

while (rs.next()) {
    // Procesar fila... el driver va trayendo lotes de 1000 según se necesitan
    procesar(rs);
}
```

| fetchSize | Comportamiento | Memoria | Rendimiento |
|---|---|---|---|
| 0 (default) | Depende del driver | Variable | Variable |
| 1-10 | Muchos round-trips | Mínima | Lento (mucha latencia) |
| 100-500 | Balance razonable | Baja | Bueno para procesamiento fila a fila |
| 1000-5000 | Pocos round-trips | Media | Bueno para transferencia de datos |
| `Integer.MIN_VALUE` | Streaming (MySQL) | Mínima | Lento pero seguro con datasets enormes |

### Caso real: cargar un CSV de 1 millón de líneas a base de datos

```java
import java.io.*;
import java.sql.*;

/**
 * Carga un archivo CSV de gran tamaño usando batch processing.
 * Estrategia: leer CSV línea por línea (streaming), acumular batches de 5000,
 * enviar a BD, commit, continuar.
 */
public class CsvToDatabaseLoader {

    private static final int BATCH_SIZE = 5000;
    private static final int LOG_INTERVAL = 100_000;

    public static void cargarCsv(String csvPath, DataSource dataSource) throws Exception {
        String sql = "INSERT INTO datos (col1, col2, col3, col4, col5) VALUES (?, ?, ?, ?, ?)";

        long inicio = System.currentTimeMillis();
        long totalLineas = 0;

        try (Connection conn = dataSource.getConnection()) {
            conn.setAutoCommit(false);

            try (BufferedReader reader = new BufferedReader(
                     new InputStreamReader(new FileInputStream(csvPath), "UTF-8"));
                 PreparedStatement ps = conn.prepareStatement(sql)) {

                String linea = reader.readLine(); // cabecera
                int batchCounter = 0;

                while ((linea = reader.readLine()) != null) {
                    String[] campos = linea.split(",", -1);

                    ps.setString(1, campos[0]);
                    ps.setInt(2, Integer.parseInt(campos[1]));
                    ps.setDouble(3, Double.parseDouble(campos[2]));
                    ps.setTimestamp(4, Timestamp.valueOf(campos[3]));
                    ps.setString(5, campos[4]);

                    ps.addBatch();
                    batchCounter++;
                    totalLineas++;

                    if (batchCounter >= BATCH_SIZE) {
                        ps.executeBatch();
                        conn.commit();
                        batchCounter = 0;
                    }

                    if (totalLineas % LOG_INTERVAL == 0) {
                        long elapsed = System.currentTimeMillis() - inicio;
                        double rate = totalLineas / (elapsed / 1000.0);
                        System.out.printf("Procesadas %d líneas (%.0f líneas/s)%n",
                                         totalLineas, rate);
                    }
                }

                if (batchCounter > 0) {
                    ps.executeBatch();
                    conn.commit();
                }
            }
        }

        long total = System.currentTimeMillis() - inicio;
        System.out.printf("Carga completada: %d líneas en %.1f segundos (%.0f líneas/s)%n",
                         totalLineas, total / 1000.0,
                         totalLineas / (total / 1000.0));
    }
}
```

### Optimización adicional: COPY / LOAD DATA INFILE

Para cargas masivas extremas (millones de filas), usa el mecanismo nativo de cada BD:

```sql
-- PostgreSQL: COPY (el más rápido, 5-10x más que JDBC batch)
COPY datos (col1, col2, col3) FROM '/ruta/datos.csv' DELIMITER ',' CSV HEADER;

-- MySQL: LOAD DATA INFILE
LOAD DATA INFILE '/ruta/datos.csv'
INTO TABLE datos
FIELDS TERMINATED BY ','
ENCLOSED BY '"'
LINES TERMINATED BY '\n'
IGNORE 1 ROWS;
```

---

## 9.12 Manejo de esquemas: Migraciones

### El problema

En desarrollo, las bases de datos evolucionan: nuevas tablas, nuevas columnas, cambios de tipo, índices. El problema surge con múltiples entornos (dev, staging, producción) y múltiples desarrolladores.

**Sin migraciones:** cada desarrollador aplica cambios a mano. Resultado: caos, entornos inconsistentes, despliegues fallidos.

```
Dev A: "Añadí columna 'telefono' a usuarios, ejecutad: ALTER TABLE..."
Dev B: "Yo cambié 'telefono' por 'phone' en mi rama..."
Producción: aún con el esquema original. Despliegue = error.
```

### Flyway: Migraciones versionadas

**Flyway** es la herramienta de migraciones más popular en Java. Es la opción por defecto en Spring Boot.

**Concepto clave:** cada cambio de esquema es un archivo SQL versionado. Flyway lleva una tabla `flyway_schema_history` para saber qué migraciones ya se aplicaron.

**Convención de nombres:**

```
V<version>__<descripcion>.sql

V1__create_users.sql
V1.1__add_email_to_users.sql
V2__create_orders.sql
V2.1__add_status_to_orders.sql
V3__create_order_items.sql
```

#### Ejemplo de migraciones Flyway

**V1__create_users.sql:**
```sql
CREATE TABLE usuarios (
    id       BIGINT AUTO_INCREMENT PRIMARY KEY,
    nombre   VARCHAR(100) NOT NULL,
    activo   BOOLEAN NOT NULL DEFAULT TRUE,
    creado_en TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

**V2__add_email_to_users.sql:**
```sql
ALTER TABLE usuarios ADD COLUMN email VARCHAR(150);
ALTER TABLE usuarios ADD CONSTRAINT uq_usuarios_email UNIQUE (email);
```

**V3__create_orders.sql:**
```sql
CREATE TABLE pedidos (
    id          BIGINT AUTO_INCREMENT PRIMARY KEY,
    usuario_id  BIGINT NOT NULL,
    total       DECIMAL(12, 2) NOT NULL,
    estado      VARCHAR(20) NOT NULL DEFAULT 'PENDIENTE',
    creado_en   TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (usuario_id) REFERENCES usuarios(id)
);

CREATE INDEX idx_pedidos_usuario ON pedidos(usuario_id);
CREATE INDEX idx_pedidos_estado ON pedidos(estado);
```

### Configuración de Flyway en un proyecto Java

**Maven:**
```xml
<dependency>
    <groupId>org.flywaydb</groupId>
    <artifactId>flyway-core</artifactId>
    <version>9.22.3</version>
</dependency>
<!-- Opcional: integración específica -->
<dependency>
    <groupId>org.flywaydb</groupId>
    <artifactId>flyway-mysql</artifactId>
    <version>9.22.3</version>
</dependency>
```

**Configuración y ejecución desde Java:**

```java
import org.flywaydb.core.Flyway;
import org.flywaydb.core.api.MigrationInfoService;
import org.flywaydb.core.api.MigrationInfo;

public class DatabaseMigration {

    public static void main(String[] args) {
        Flyway flyway = Flyway.configure()
            .dataSource("jdbc:mysql://localhost:3306/mi_app", "admin", "secreto")
            .locations("classpath:db/migration")  // Carpeta donde están los .sql
            .table("schema_history")              // Nombre tabla de control
            .baselineOnMigrate(true)              // Si BD ya existe, marcar baseline
            .validateMigrationNaming(true)        // Validar nombres de archivo
            .load();

        // Ejecutar migraciones pendientes
        int aplicadas = flyway.migrate();
        System.out.println("Migraciones aplicadas: " + aplicadas);

        // Verificar estado
        MigrationInfoService info = flyway.info();
        System.out.println("Pendientes: " + info.pending().length);
        System.out.println("Aplicadas: " + info.applied().length);
    }
}
```

**Estructura del proyecto:**

```
src/
  main/
    resources/
      db/
        migration/
          V1__create_users.sql
          V2__add_email_to_users.sql
          V3__create_orders.sql
```

### Comandos útiles de Flyway

| Comando | Descripción |
|---|---|
| `migrate()` | Aplica todas las migraciones pendientes. |
| `info()` | Muestra estado: aplicadas, pendientes, fallidas. |
| `validate()` | Valida que migraciones aplicadas coincidan con archivos. |
| `repair()` | Repara la tabla de historial si una migración falló parcialmente. |
| `undo()` | Revierte la última migración (requiere migraciones de undo: `U1__...`). |
| `baseline()` | Marca la BD actual como baseline para empezar a usar Flyway. |
| `clean()` | Borra todos los objetos de la BD. Desactivar en producción. |

### Reparación de migraciones fallidas

```java
// Ver qué falló
MigrationInfo[] failed = flyway.info().failed();
for (MigrationInfo f : failed) {
    System.out.println("Fallida: " + f.getDescription());
}

// Reparar
flyway.repair();

// Reintentar
flyway.migrate();
```

### Mejores prácticas con Flyway

1. **Nunca modifiques una migración ya aplicada.** Si `V2__add_email.sql` está en producción, no lo modifiques. Crea `V5__change_email_length.sql`.

2. **Siempre forward-only.** Prefiere migraciones que solo avancen. Las migraciones undo son propensas a errores.

3. **Incluye migraciones en el pipeline CI/CD:**

```java
@Configuration
public class FlywayConfig {
    @Bean
    public Flyway flyway(DataSource dataSource) {
        return Flyway.configure()
            .dataSource(dataSource)
            .locations("classpath:db/migration")
            .load();
    }

    @EventListener(ApplicationReadyEvent.class)
    public void migrate() {
        flyway(dataSource()).migrate();
    }
}
```

4. **Usa migraciones repetibles para datos de referencia:**

```
R__seed_categories.sql   # Se ejecuta cada vez que cambia
```

Las migraciones `R__` se re-ejecutan si cambia su checksum. Útiles para datos semilla: categorías, países, permisos.

### Liquibase: Alternativa a Flyway

**Liquibase** es más flexible pero más complejo. Soporta XML, YAML, JSON y SQL. Ofrece rollback automático para cambios simples.

**Ejemplo en YAML (changelog.yaml):**

```yaml
databaseChangeLog:
  - changeSet:
      id: 1
      author: andres
      changes:
        - createTable:
            tableName: usuarios
            columns:
              - column:
                  name: id
                  type: BIGINT
                  autoIncrement: true
                  constraints:
                    primaryKey: true
              - column:
                  name: nombre
                  type: VARCHAR(100)
              - column:
                  name: email
                  type: VARCHAR(150)
```

**Ejemplo en SQL con rollback:**

```sql
-- liquibase formatted sql

-- changeset andres:2
ALTER TABLE usuarios ADD COLUMN telefono VARCHAR(20);
-- rollback ALTER TABLE usuarios DROP COLUMN telefono;
```

**Configuración de Liquibase desde Java:**

```java
import liquibase.Liquibase;
import liquibase.database.DatabaseFactory;
import liquibase.database.jvm.JdbcConnection;
import liquibase.resource.ClassLoaderResourceAccessor;

Liquibase liquibase = new Liquibase(
    "db/changelog.yaml",
    new ClassLoaderResourceAccessor(),
    DatabaseFactory.getInstance().findCorrectDatabaseImplementation(
        new JdbcConnection(dataSource.getConnection())
    )
);

liquibase.update(""); // Aplica migraciones pendientes
```

### ¿Flyway o Liquibase?

| Característica | Flyway | Liquibase |
|---|---|---|
| Simplicidad | ★★★★★ | ★★★☆☆ |
| Rollback | Manual | Automático (cambios simples) |
| Formatos | SQL | SQL, XML, YAML, JSON |
| Curva de aprendizaje | Mínima | Media |
| Integración Spring Boot | Nativa | Nativa |
| Adecuado para | Equipos que prefieren SQL directo | Equipos con multi-BD o rollback automático |

---

## 9.13 Testing de acceso a datos

Probar código que accede a BD es un desafío: necesitas una BD disponible, datos predecibles y aislamiento entre tests.

### H2 en memoria para tests unitarios

H2 es una base de datos Java pura que puede ejecutarse en memoria. Ideal para tests rápidos y aislados.

```xml
<dependency>
    <groupId>com.h2database</groupId>
    <artifactId>h2</artifactId>
    <version>2.2.224</version>
    <scope>test</scope>
</dependency>
```

```java
import com.zaxxer.hikari.HikariConfig;
import com.zaxxer.hikari.HikariDataSource;
import org.junit.jupiter.api.*;

import java.sql.*;
import java.util.Optional;

import static org.junit.jupiter.api.Assertions.*;

@TestInstance(TestInstance.Lifecycle.PER_CLASS)
class UsuarioDAOTest {

    private HikariDataSource dataSource;
    private UsuarioDAO usuarioDAO;

    @BeforeAll
    void setUpAll() {
        HikariConfig config = new HikariConfig();
        config.setJdbcUrl("jdbc:h2:mem:test_usuarios;DB_CLOSE_DELAY=-1");
        config.setUsername("sa");
        config.setPassword("");
        dataSource = new HikariDataSource(config);

        try (Connection conn = dataSource.getConnection();
             Statement stmt = conn.createStatement()) {
            stmt.execute("""
                CREATE TABLE usuarios (
                    id       BIGINT AUTO_INCREMENT PRIMARY KEY,
                    nombre   VARCHAR(100) NOT NULL,
                    email    VARCHAR(150) NOT NULL UNIQUE,
                    activo   BOOLEAN NOT NULL DEFAULT TRUE,
                    creado_en TIMESTAMP DEFAULT CURRENT_TIMESTAMP
                )
                """);
        } catch (SQLException e) {
            throw new RuntimeException(e);
        }

        usuarioDAO = new UsuarioDAO(dataSource);
    }

    @AfterEach
    void tearDown() {
        try (Connection conn = dataSource.getConnection();
             Statement stmt = conn.createStatement()) {
            stmt.execute("DELETE FROM usuarios");
        } catch (SQLException e) {
            e.printStackTrace();
        }
    }

    @AfterAll
    void tearDownAll() {
        dataSource.close();
    }

    @Test
    void testSaveAndFindById() {
        Usuario u = new Usuario();
        u.setNombre("Test User");
        u.setEmail("test@example.com");
        u.setActivo(true);

        Usuario saved = usuarioDAO.save(u);

        assertNotNull(saved.getId());
        assertTrue(saved.getId() > 0);

        Optional<Usuario> found = usuarioDAO.findById(saved.getId());
        assertTrue(found.isPresent());
        assertEquals("Test User", found.get().getNombre());
    }

    @Test
    void testFindByIdNotFound() {
        Optional<Usuario> found = usuarioDAO.findById(9999L);
        assertTrue(found.isEmpty());
    }

    @Test
    void testSaveWithDuplicateEmail() {
        Usuario u1 = new Usuario();
        u1.setNombre("User 1");
        u1.setEmail("dup@example.com");
        usuarioDAO.save(u1);

        Usuario u2 = new Usuario();
        u2.setNombre("User 2");
        u2.setEmail("dup@example.com");

        assertThrows(RuntimeException.class, () -> usuarioDAO.save(u2));
    }

    @Test
    void testUpdate() {
        Usuario u = new Usuario();
        u.setNombre("Original");
        u.setEmail("original@example.com");
        Usuario saved = usuarioDAO.save(u);

        saved.setNombre("Modificado");
        usuarioDAO.update(saved);

        Optional<Usuario> updated = usuarioDAO.findById(saved.getId());
        assertTrue(updated.isPresent());
        assertEquals("Modificado", updated.get().getNombre());
    }

    @Test
    void testDeleteById() {
        Usuario u = new Usuario();
        u.setNombre("To Delete");
        u.setEmail("delete@example.com");
        Usuario saved = usuarioDAO.save(u);

        usuarioDAO.deleteById(saved.getId());
        assertTrue(usuarioDAO.findById(saved.getId()).isEmpty());
    }

    @Test
    void testSearchByNombre() {
        usuarioDAO.save(crearUsuario("Andrés García", "andres@ex.com"));
        usuarioDAO.save(crearUsuario("María García", "maria@ex.com"));
        usuarioDAO.save(crearUsuario("Carlos López", "carlos@ex.com"));

        var resultados = usuarioDAO.searchByNombre("García");
        assertEquals(2, resultados.size());
    }

    private Usuario crearUsuario(String nombre, String email) {
        Usuario u = new Usuario();
        u.setNombre(nombre);
        u.setEmail(email);
        u.setActivo(true);
        return u;
    }
}
```

### Principio: cada test debe ser independiente

Los tests **no deben compartir estado de BD**. Cada test debe preparar sus propios datos, ejecutar sus aserciones y limpiar lo creado (o se limpia todo en `@AfterEach`).

**ANTIPATRÓN:** tests que dependen de datos creados por otros tests.

```java
// MAL: testUpdate asume que testSave creó id=1
@Test void testSave() { ... }
@Test void testUpdate() {
    Usuario u = usuarioDAO.findById(1L).get();
}
```

**PATRÓN CORRECTO:**

```java
@Test void testUpdate() {
    // Cada test crea sus propios datos
    Usuario u = new Usuario();
    u.setNombre("Para Update");
    u.setEmail("update@test.com");
    Usuario saved = usuarioDAO.save(u);

    saved.setNombre("Actualizado");
    usuarioDAO.update(saved);
    assertEquals("Actualizado", usuarioDAO.findById(saved.getId()).get().getNombre());
}
```

### Testcontainers: base de datos real en Docker

H2 es útil para tests unitarios, pero tiene diferencias con PostgreSQL/MySQL reales (funciones SQL, tipos de datos, comportamiento de índices). Para tests de integración realistas, usa **Testcontainers**.

```xml
<dependency>
    <groupId>org.testcontainers</groupId>
    <artifactId>testcontainers</artifactId>
    <version>1.19.3</version>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>org.testcontainers</groupId>
    <artifactId>postgresql</artifactId>
    <version>1.19.3</version>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>org.testcontainers</groupId>
    <artifactId>junit-jupiter</artifactId>
    <version>1.19.3</version>
    <scope>test</scope>
</dependency>
```

```java
import com.zaxxer.hikari.HikariConfig;
import com.zaxxer.hikari.HikariDataSource;
import org.junit.jupiter.api.*;
import org.testcontainers.containers.PostgreSQLContainer;
import org.testcontainers.junit.jupiter.Container;
import org.testcontainers.junit.jupiter.Testcontainers;

import java.sql.*;
import java.util.Optional;

import static org.junit.jupiter.api.Assertions.*;

@Testcontainers
@TestInstance(TestInstance.Lifecycle.PER_CLASS)
class UsuarioDAOIntegrationTest {

    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16")
            .withDatabaseName("testdb")
            .withUsername("testuser")
            .withPassword("testpass");

    private HikariDataSource dataSource;
    private UsuarioDAO usuarioDAO;

    @BeforeAll
    void setUpAll() {
        HikariConfig config = new HikariConfig();
        config.setJdbcUrl(postgres.getJdbcUrl());
        config.setUsername(postgres.getUsername());
        config.setPassword(postgres.getPassword());
        config.setMaximumPoolSize(5);
        dataSource = new HikariDataSource(config);

        try (Connection conn = dataSource.getConnection();
             Statement stmt = conn.createStatement()) {
            stmt.execute("""
                CREATE TABLE IF NOT EXISTS usuarios (
                    id       BIGSERIAL PRIMARY KEY,
                    nombre   VARCHAR(100) NOT NULL,
                    email    VARCHAR(150) NOT NULL UNIQUE,
                    activo   BOOLEAN NOT NULL DEFAULT TRUE,
                    creado_en TIMESTAMP DEFAULT CURRENT_TIMESTAMP
                )
                """);
        } catch (SQLException e) {
            throw new RuntimeException("Error creando esquema", e);
        }

        usuarioDAO = new UsuarioDAO(dataSource);
    }

    @AfterEach
    void tearDown() {
        try (Connection conn = dataSource.getConnection();
             Statement stmt = conn.createStatement()) {
            stmt.execute("DELETE FROM usuarios");
        } catch (SQLException e) {
            e.printStackTrace();
        }
    }

    @AfterAll
    void tearDownAll() {
        dataSource.close();
    }

    @Test
    void testCrudCompletoConPostgreSQL() {
        Usuario u = new Usuario();
        u.setNombre("Integration User");
        u.setEmail("integration@example.com");
        u.setActivo(true);

        Usuario saved = usuarioDAO.save(u);
        assertNotNull(saved.getId());

        Optional<Usuario> found = usuarioDAO.findById(saved.getId());
        assertTrue(found.isPresent());
        assertEquals("Integration User", found.get().getNombre());

        saved.setNombre("Updated");
        usuarioDAO.update(saved);
        found = usuarioDAO.findById(saved.getId());
        assertEquals("Updated", found.get().getNombre());

        usuarioDAO.deleteById(saved.getId());
        assertTrue(usuarioDAO.findById(saved.getId()).isEmpty());
    }
}
```

### DBUnit: cargar datasets de prueba

**DBUnit** permite cargar conjuntos de datos predefinidos (XML, CSV) antes de cada test:

```xml
<dependency>
    <groupId>org.dbunit</groupId>
    <artifactId>dbunit</artifactId>
    <version>2.7.3</version>
    <scope>test</scope>
</dependency>
```

**Archivo de dataset (usuarios-dataset.xml):**
```xml
<?xml version="1.0" encoding="UTF-8"?>
<dataset>
    <usuarios id="1" nombre="Andrés" email="andres@test.com" activo="true" />
    <usuarios id="2" nombre="María" email="maria@test.com" activo="false" />
    <usuarios id="3" nombre="Carlos" email="carlos@test.com" activo="true" />
</dataset>
```

```java
import org.dbunit.*;
import org.dbunit.database.DatabaseConnection;
import org.dbunit.database.IDatabaseConnection;
import org.dbunit.dataset.IDataSet;
import org.dbunit.dataset.xml.FlatXmlDataSetBuilder;
import org.dbunit.operation.DatabaseOperation;
import org.junit.jupiter.api.*;

class UsuarioDAODBUnitTest {

    private static HikariDataSource dataSource;
    private UsuarioDAO usuarioDAO;
    private IDatabaseConnection dbUnitConnection;

    @BeforeAll
    static void setUpAll() {
        HikariConfig config = new HikariConfig();
        config.setJdbcUrl("jdbc:h2:mem:dbunit_test;DB_CLOSE_DELAY=-1");
        config.setUsername("sa");
        config.setPassword("");
        dataSource = new HikariDataSource(config);

        try (Connection conn = dataSource.getConnection();
             Statement stmt = conn.createStatement()) {
            stmt.execute("""
                CREATE TABLE usuarios (
                    id       BIGINT AUTO_INCREMENT PRIMARY KEY,
                    nombre   VARCHAR(100) NOT NULL,
                    email    VARCHAR(150) NOT NULL UNIQUE,
                    activo   BOOLEAN NOT NULL DEFAULT TRUE,
                    creado_en TIMESTAMP DEFAULT CURRENT_TIMESTAMP
                )
                """);
        } catch (SQLException e) {
            throw new RuntimeException(e);
        }
    }

    @BeforeEach
    void setUp() throws Exception {
        usuarioDAO = new UsuarioDAO(dataSource);

        Connection jdbcConn = dataSource.getConnection();
        dbUnitConnection = new DatabaseConnection(jdbcConn);

        IDataSet dataset = new FlatXmlDataSetBuilder().build(
            getClass().getResourceAsStream("/datasets/usuarios-dataset.xml")
        );

        DatabaseOperation.CLEAN_INSERT.execute(dbUnitConnection, dataset);
    }

    @AfterEach
    void tearDown() throws Exception {
        dbUnitConnection.close();
    }

    @AfterAll
    static void tearDownAll() {
        dataSource.close();
    }

    @Test
    void testFindByIdConDataset() {
        Optional<Usuario> found = usuarioDAO.findById(1L);
        assertTrue(found.isPresent());
        assertEquals("Andrés", found.get().getNombre());
    }

    @Test
    void testFindByActivoConDataset() {
        var activos = usuarioDAO.findByActivo(true);
        assertEquals(2, activos.size());
    }

    @Test
    void testCountConDataset() {
        assertEquals(3, usuarioDAO.count());
    }
}
```

### Rollback después de cada test

Ejecutar cada test dentro de una transacción y hacer rollback al final. Spring lo hace automáticamente con `@Transactional` en tests. En JDBC puro:

```java
class TransactionalTest {

    private DataSource dataSource;
    private Connection testConnection;

    @BeforeEach
    void setUp() throws SQLException {
        testConnection = dataSource.getConnection();
        testConnection.setAutoCommit(false);
    }

    @AfterEach
    void tearDown() throws SQLException {
        testConnection.rollback();
        testConnection.close();
    }

    @Test
    void testQueModificaDatos() throws SQLException {
        PreparedStatement ps = testConnection.prepareStatement(
            "INSERT INTO usuarios (nombre, email) VALUES (?, ?)");
        ps.setString(1, "Temporal");
        ps.setString(2, "temp@test.com");
        ps.executeUpdate();

        Statement stmt = testConnection.createStatement();
        ResultSet rs = stmt.executeQuery(
            "SELECT COUNT(*) FROM usuarios WHERE email='temp@test.com'");
        rs.next();
        assertEquals(1, rs.getInt(1));

        // Al salir, rollback() → el registro no persiste
    }
}
```

### Resumen de estrategias de testing

| Estrategia | Velocidad | Aislamiento | Realismo | Setup |
|---|---|---|---|---|
| H2 en memoria | ★★★★★ | ★★★★★ | ★★☆☆☆ | Mínimo |
| Testcontainers PostgreSQL | ★★★☆☆ | ★★★★★ | ★★★★★ | Docker |
| DBUnit datasets | ★★★★☆ | ★★★★☆ | ★★★☆☆ | XML/CSV |
| Rollback por test | ★★★★★ | ★★★★★ | ★★★★☆ | Manual |
| BD compartida | ★★☆☆☆ | ★☆☆☆☆ | ★★★★★ | Evitar |
---

## 9.14 Conceptos de ORM: ¿y después de JDBC?

### JDBC como base: por qué es importante entenderlo

JDBC es la capa fundamental sobre la que se construyen todos los frameworks de acceso a datos en Java. JPA, Hibernate, Spring Data JPA, MyBatis, jOOQ... todos usan JDBC internamente.

**Saber JDBC te permite:**

- **Depurar problemas de rendimiento**: cuando Hibernate genera 100 queries en lugar de 1 (problema N+1), identificar el problema requiere entender SQL y JDBC.
- **Escapar del ORM cuando es necesario**: hay consultas que ningún ORM puede generar bien.
- **Escribir operaciones batch y ETL**: donde JDBC puro es 10-50x más rápido que un ORM.
- **Entender qué ocurre en producción**: los logs de consultas lentas muestran SQL, no llamadas a `entityManager.find()`.

### JPA (Jakarta Persistence API): El estándar de ORM

JPA es la especificación estándar para ORM en Java. Define interfaces y anotaciones. No es una implementación; es un contrato.

**Anotaciones principales de JPA:**

```java
@Entity
@Table(name = "usuarios")
public class Usuario {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "nombre", length = 100, nullable = false)
    private String nombre;

    @Column(name = "email", unique = true)
    private String email;

    @Column(name = "activo")
    private Boolean activo = true;

    @Column(name = "creado_en")
    private LocalDateTime creadoEn;
}
```

### Hibernate: La implementación más popular

Hibernate es la implementación de JPA más usada. Spring Boot lo incluye por defecto a través de Spring Data JPA, que a su vez usa Hibernate.

### La misma consulta: JDBC vs JPA (EntityManager) vs Spring Data JPA

**JDBC (puro, manual):**

```java
public Optional<Usuario> findById(Long id) {
    String sql = "SELECT id, nombre, email, activo, creado_en FROM usuarios WHERE id = ?";
    try (Connection conn = dataSource.getConnection();
         PreparedStatement ps = conn.prepareStatement(sql)) {
        ps.setLong(1, id);
        ResultSet rs = ps.executeQuery();
        if (rs.next()) {
            return Optional.of(mapRow(rs));
        }
        return Optional.empty();
    } catch (SQLException e) {
        throw new RuntimeException(e);
    }
}

private Usuario mapRow(ResultSet rs) throws SQLException {
    Usuario u = new Usuario();
    u.setId(rs.getLong("id"));
    u.setNombre(rs.getString("nombre"));
    u.setEmail(rs.getString("email"));
    u.setActivo(rs.getBoolean("activo"));
    Timestamp ts = rs.getTimestamp("creado_en");
    if (ts != null) u.setCreadoEn(ts.toLocalDateTime());
    return u;
}
```

**JPA (EntityManager):**

```java
@Repository
public class UsuarioRepositoryJPA {

    @PersistenceContext
    private EntityManager em;

    public Optional<Usuario> findById(Long id) {
        return Optional.ofNullable(em.find(Usuario.class, id));
    }

    public void save(Usuario usuario) {
        em.persist(usuario);
    }

    public List<Usuario> findByActivo(boolean activo) {
        return em.createQuery(
            "SELECT u FROM Usuario u WHERE u.activo = :activo", Usuario.class)
            .setParameter("activo", activo)
            .getResultList();
    }

    public List<Usuario> searchByNombre(String keyword) {
        return em.createQuery(
            "SELECT u FROM Usuario u WHERE u.nombre LIKE :keyword", Usuario.class)
            .setParameter("keyword", "%" + keyword + "%")
            .getResultList();
    }
}
```

**Spring Data JPA (más abstracto aún):**

```java
public interface UsuarioRepository extends JpaRepository<Usuario, Long> {
    Optional<Usuario> findByEmail(String email);
    List<Usuario> findByActivo(boolean activo);
    List<Usuario> findByNombreContaining(String keyword);
    long countByActivo(boolean activo);
}
```

Cero implementación. Spring Data genera el SQL automáticamente basándose en el nombre del método.

### Ventajas de JDBC directo

| Ventaja | Descripción |
|---|---|
| **Control total** | Escribes exactamente el SQL que quieres. Sin sorpresas. |
| **Rendimiento predecible** | Sabes cuántas queries se ejecutan. No hay lazy loading oculto. |
| **Sin magia** | No hay proxies, sesiones, estados "managed" vs "detached". |
| **Mejor para batch/ETL** | JDBC batch es mucho más rápido que `saveAll()` de JPA. |
| **Menor consumo de memoria** | No hay contexto de persistencia (L1 cache) acumulando objetos. |
| **SQL complejo** | CTEs, window functions, pivots... escribir esto en JPQL es una pesadilla. |

### Ventajas de JPA/Hibernate

| Ventaja | Descripción |
|---|---|
| **Productividad** | Menos código boilerplate. CRUD automático con Spring Data. |
| **Caché L1 (sesión)** | Misma entidad leída dos veces en la misma sesión = una sola query. |
| **Caché L2 (compartida)** | Caché entre sesiones. Reduce consultas a BD. |
| **Dirty checking** | Hibernate detecta cambios en entidades y genera UPDATEs automáticos. |
| **Migración de motor de BD** | Cambiar de MySQL a PostgreSQL es cambiar un dialecto. |
| **Lazy loading** | Relaciones que se cargan solo cuando se acceden. |
| **Optimistic locking** | `@Version` para control de concurrencia sin locks. |

### Cuándo usar cada tecnología

| Tecnología | Cuándo usarla |
|---|---|
| **JDBC puro** | ETL, batch processing, migraciones de datos, consultas muy complejas, apps pequeñas. |
| **Spring JdbcTemplate** | Menos boilerplate que JDBC puro pero control total del SQL. Buen balance. |
| **JPA / Hibernate** | CRUD estándar, relaciones complejas entre entidades, apps empresariales. |
| **Spring Data JPA** | Máxima productividad en operaciones CRUD. Sobre JPA/Hibernate. |
| **jOOQ** | SQL type-safe generado desde la BD. Ideal para SQL complejo con seguridad de tipos. |
| **MyBatis** | Escribes tu propio SQL pero con mapping automático ResultSet → Objeto. |

### Ejemplo comparativo: misma operación en 4 frameworks

**Consulta:** "Buscar usuarios cuyo nombre contenga 'García', ordenados por creado_en descendente, máximo 10 resultados."

**JDBC puro:**
```java
String sql = "SELECT id, nombre, email, activo, creado_en FROM usuarios " +
             "WHERE nombre LIKE ? ORDER BY creado_en DESC LIMIT 10";
try (Connection conn = ds.getConnection();
     PreparedStatement ps = conn.prepareStatement(sql)) {
    ps.setString(1, "%García%");
    ResultSet rs = ps.executeQuery();
    List<Usuario> result = new ArrayList<>();
    while (rs.next()) { result.add(mapRow(rs)); }
    return result;
}
```

**JdbcTemplate (Spring):**
```java
return jdbcTemplate.query(
    "SELECT id, nombre, email, activo, creado_en FROM usuarios " +
    "WHERE nombre LIKE ? ORDER BY creado_en DESC LIMIT 10",
    new UsuarioRowMapper(), "%García%");
```

**JPA (Hibernate):**
```java
return em.createQuery(
    "SELECT u FROM Usuario u WHERE u.nombre LIKE :nombre ORDER BY u.creadoEn DESC",
    Usuario.class)
    .setParameter("nombre", "%García%")
    .setMaxResults(10)
    .getResultList();
```

**jOOQ (type-safe):**
```java
return dsl.selectFrom(USUARIOS)
    .where(USUARIOS.NOMBRE.like("%García%"))
    .orderBy(USUARIOS.CREADO_EN.desc())
    .limit(10)
    .fetchInto(Usuario.class);
```

---

## 9.15 Mejores prácticas de base de datos en producción

### 1. Siempre usar PreparedStatement

Nunca concatenes datos de usuario en SQL. No es una recomendación, es una **regla de seguridad**. La inyección SQL sigue siendo la vulnerabilidad #1 en aplicaciones web según OWASP.

```java
// PROHIBIDO
String query = "SELECT * FROM users WHERE username = '" + input + "'";

// OBLIGATORIO
String query = "SELECT * FROM users WHERE username = ?";
ps.setString(1, input);
```

### 2. Establecer timeouts

Sin timeouts, una consulta lenta puede bloquear un hilo del pool indefinidamente:

```java
// Timeout por consulta (segundos)
ps.setQueryTimeout(30);

// Timeout del socket JDBC
config.addDataSourceProperty("socketTimeout", "30000");

// Timeout de obtención de conexión del pool
config.setConnectionTimeout(10_000);
config.setValidationTimeout(5_000);

// Timeout de transacción (motor específico)
stmt.execute("SET lock_timeout = '10s'");
```

**¿Qué pasa sin timeouts?**

```
[pool-1-thread-3] Ejecutando SELECT ... WHERE ... (bloqueado por un lock)
→ El hilo se bloquea para siempre
→ La conexión del pool nunca se libera
→ 10 minutos después, 50 hilos bloqueados, pool agotado
→ La aplicación deja de responder
```

### 3. Retry lógico para deadlocks

En sistemas con alta concurrencia, los deadlocks son inevitables. La estrategia es **reintentarlos** con backoff exponencial.

Ya vimos la implementación completa en la Sección 9.6 (Transacciones y niveles de aislamiento). En resumen:

```java
public <T> T ejecutarConRetry(OperacionTransaccional<T> operacion) throws SQLException {
    for (int intento = 0; intento < MAX_RETRIES; intento++) {
        try (Connection conn = dataSource.getConnection()) {
            conn.setAutoCommit(false);
            T result = operacion.ejecutar(conn);
            conn.commit();
            return result;
        } catch (SQLException e) {
            if (!esDeadlock(e) || intento == MAX_RETRIES - 1) throw e;
            Thread.sleep(RETRY_DELAY_MS * (long) Math.pow(2, intento + 1));
        }
    }
    throw new SQLException("Agotados reintentos por deadlock");
}
```

### 4. Monitoring: métricas de pool, queries lentas, conexiones activas

Monitorizar el pool es esencial en producción:

```java
HikariPoolMXBean pool = ds.getHikariPoolMXBean();
int activas = pool.getActiveConnections();
int pendientes = pool.getThreadsAwaitingConnection();

// Loggear periódicamente
if (activas > maximumPoolSize * 0.8) {
    logger.warn("Pool al {}% de capacidad", activas * 100 / maximumPoolSize);
}
if (pendientes > 0) {
    logger.error("{} hilos esperando conexión", pendientes);
}
```

**Métricas clave a vigilar:**

- **Active connections**: si se acerca al máximo, ampliar pool o investigar queries lentas.
- **Pending threads**: si > 0, los usuarios experimentan latencia.
- **Connection timeout rate**: si > 0%, la aplicación falla.
- **Slow query log**: habilitar en la BD para detectar queries > 1 segundo.
- **Deadlock rate**: si es alto, revisar orden de locks en transacciones.

### 5. Health checks: validar conexiones

Validar conexiones antes de entregarlas evita errores por conexiones rotas:

```java
// Configuración recomendada en HikariCP
config.setConnectionTestQuery("SELECT 1");  // Query ligera de validación
config.setValidationTimeout(5_000);         // 5 segundos máximo para validar
config.setMaxLifetime(1_200_000);           // 20 min — reciclar antes que timeout del servidor

// JDBC 4: isValid() es más rápido (el driver puede validar sin query)
// Pero connectionTestQuery es más confiable en entornos con balanceadores
```

### 6. Nunca guardar contraseñas en texto plano

Usa variables de entorno, archivos de configuración externos o secret managers:

```java
// Variables de entorno (recomendado para contenedores/K8s)
config.setUsername(System.getenv("DB_USERNAME"));
config.setPassword(System.getenv("DB_PASSWORD"));

// Archivo externo (no dentro del JAR)
Properties props = new Properties();
try (InputStream fis = Files.newInputStream(Paths.get("/etc/app/db.properties"))) {
    props.load(fis);
}
config.setPassword(props.getProperty("db.password"));

// HashiCorp Vault / AWS Secrets Manager
String password = vaultClient.getSecret("database/prod").get("password");
config.setPassword(password);
```

### 7. Preferir nombres de columna sobre índices en ResultSet

```java
// Menos mantenible: ¿qué columna es la posición 2?
rs.getString(2);

// Más mantenible: explícito y resistente a cambios en SELECT
rs.getString("nombre");

// Rendimiento: la diferencia es insignificante en la práctica.
// La claridad y mantenibilidad superan cualquier micro-optimización.
```

Si necesitas rendimiento extremo, usa `getInt(1)` solo en bucles muy calientes y con `ResultSetMetaData` para verificar.

### 8. Cifrar la conexión (SSL/TLS)

```
jdbc:mysql://host:3306/db?useSSL=true&requireSSL=true
jdbc:postgresql://host:5432/db?sslmode=require
```

Siempre activo en producción. Sin SSL, las credenciales y datos viajan en texto plano por la red.

### 9. Validar datos antes de enviarlos a la BD

```java
// Validar en el servicio, antes de llamar al DAO
if (email == null || !email.contains("@")) {
    throw new IllegalArgumentException("Email inválido");
}
if (nombre.length() > 100) {
    throw new IllegalArgumentException("Nombre demasiado largo");
}
```

Capturar errores temprano evita excepciones SQL confusas y tráfico innecesario a la BD.

### 10. Manejo adecuado de excepciones

```java
try {
    // operaciones JDBC...
} catch (SQLException e) {
    // Loggear para debugging interno
    logger.error("Error de BD [SQLState={}, Code={}]", e.getSQLState(), e.getErrorCode(), e);
    // Lanzar excepción de aplicación sin detalles internos
    throw new DataAccessException("Error al acceder a los datos. Intente nuevamente.", e);
}
```

Nunca expongas mensajes de error SQL al usuario final. Contienen información sobre la estructura de tu BD.

---

## Ejercicios propuestos

1. **ProductoDAO completo.** Implementa un `ProductoDAO` (tabla `productos`) con CRUD completo usando el patrón DAO. Incluye búsqueda por nombre parcial y por rango de precio. Escribe tests unitarios con H2.

2. **Transferencia bancaria con transacciones.** Escribe un programa que transfiera dinero entre dos cuentas usando `setAutoCommit(false)`, `commit()` y `rollback()`. Incluye savepoints intermedios y simula fallos.

3. **Test de integración con Testcontainers.** Crea un test de integración con PostgreSQL via Testcontainers que pruebe todos los métodos del `UsuarioDAO`. Compara los resultados con H2.

4. **Batch benchmark.** Implementa el benchmark de batch y mide: 10 000 inserts uno a uno vs batch de 100, 500, 1000, 5000. Documenta los speedups obtenidos.

5. **Procedimiento almacenado.** Escribe un procedimiento llamado `calcular_estadisticas` con parámetros OUT (total_usuarios, promedio_edad) y un ResultSet. Invócalo con `CallableStatement`.

6. **Configuración de Flyway.** Configura Flyway en un proyecto nuevo. Crea migraciones V1, V2, V3 para construir un esquema completo. Prueba migrate, info, validate y repair.

7. **Simulación de deadlock y retry.** Crea un programa con 2 hilos que cause un deadlock intencionado. Implementa la lógica de retry con backoff exponencial. Mide cuántos reintentos son necesarios.

8. **Carga de CSV masivo.** Implementa un cargador de CSV a BD usando batch processing. Prueba con un archivo de 100 000 líneas y mide el rendimiento.

9. **Comparativa ORM.** Implementa la misma consulta compleja (con JOINs, filtros, ordenación, paginación) en JDBC, JPA Criteria API, y jOOQ. Compara legibilidad y rendimiento.

10. **Monitoreo de pool.** Configura HikariCP con métricas Prometheus. Crea un dashboard en Grafana que muestre: conexiones activas, inactivas, pendientes, timeouts.

---

## Resumen

- **JDBC** es la API fundamental de Java para bases de datos relacionales. Aunque existen frameworks de más alto nivel (JPA, Hibernate, Spring JDBC, MyBatis), comprender JDBC es esencial para entender cómo funcionan por debajo.

- **PreparedStatement** es obligatorio para cualquier consulta con parámetros. No solo por seguridad (previene SQL injection), sino también por rendimiento.

- **try-with-resources** garantiza que los recursos se liberen correctamente incluso en caso de excepciones.

- **HikariCP** es el pool de conexiones más rápido del ecosistema Java. Configúralo correctamente: dimensiona el pool según la fórmula `((core_count * 2) + effective_spindle_count)`, activa `leakDetectionThreshold` y monitorea las métricas en producción.

- Las **transacciones** permiten mantener la integridad de los datos. Comprende los niveles de aislamiento (READ_UNCOMMITTED, READ_COMMITTED, REPEATABLE_READ, SERIALIZABLE) y qué anomalías evitas con cada uno. Implementa retry para deadlocks.

- El patrón **DAO** encapsula el acceso a datos y separa responsabilidades, facilitando pruebas unitarias con mocks y futuros cambios de tecnología de persistencia.

- **Batch operations** (`addBatch()` + `executeBatch()`) son 20-100x más rápidas que inserciones individuales. Úsalas siempre para operaciones masivas. Controla `fetchSize` para evitar OOM con datasets grandes.

- Las **migraciones** (Flyway, Liquibase) son esenciales para evolucionar el esquema de BD de forma controlada. Nunca modifiques una migración ya aplicada. Inclúyelas en tu pipeline CI/CD.

- **Testing de acceso a datos** requiere una estrategia: H2 para tests unitarios rápidos, Testcontainers para integración realista, DBUnit para datasets predefinidos. Cada test debe ser independiente.

- Los **ORM** (JPA, Hibernate) construyen sobre JDBC. JDBC te da control total y rendimiento predecible; JPA te da productividad y abstracción. La elección depende del caso de uso. Conocer ambos es lo que diferencia a un novato de un experto.

- Los **metadatos** (`DatabaseMetaData`, `ResultSetMetaData`) son útiles para herramientas, generación dinámica de consultas y frameworks ORM.

- **RowSet** (especialmente `CachedRowSet`) ofrece una alternativa a `ResultSet` con capacidades de desconexión, navegación libre y serialización.

- En producción: **timeouts** en todas las capas, **retry** para deadlocks, **monitoreo** del pool, **validación de conexiones**, **SSL/TLS** obligatorio, y **nunca** credenciales en el código fuente.

---

← [Capítulo anterior](capitulo-08-streams-lambdas.md) | [Inicio](../README.md) | [Capítulo siguiente →](capitulo-10-buenas-practicas.md)
