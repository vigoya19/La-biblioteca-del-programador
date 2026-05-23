# Capítulo 15: Cheat Sheet y Ejercicios Resueltos Paso a Paso

> "La maestría técnica en ingeniería de bases de datos relacionales no se adquiere memorizando pasivamente la teoría. Se consolida en el asfalto práctico: resolviendo problemas reales, depurando cuellos de botella microscópicos y modelando esquemas bajo la presión de la concurrencia."

¡Felicidades por llegar al último volumen de este libro! Has recorrido un camino espectacular: desde comprender el álgebra relacional profunda y los internals físicos del motor hasta estructurar transacciones ACID estrictas, aislar datos concurrentes con RLS, optimizar consultas con `EXPLAIN ANALYZE` y diseñar migraciones zero-downtime en caliente.

Como apéndice y guía definitiva de consulta rápida, este capítulo está estructurado en dos partes:
1. **La Cheat Sheet Definitiva**: Un compendio de sintaxis SQL premium lista para copiar y pegar en tus proyectos de producción.
2. **El Mapa de Desafíos**: Una serie de ejercicios resueltos paso a paso desde el nivel **Fácil** hasta el nivel **Experto** para poner a prueba tu destreza en SQL.

---

## 15.1 La Cheat Sheet Relacional Premium

### 1. Transacciones y Aislamiento Estricto
```sql
-- Configurar aislamientoSerializable y reintentar transacciones
BEGIN TRANSACTION ISOLATION LEVEL SERIALIZABLE;
-- ... consultas ...
COMMIT;
```

### 2. Bloqueos de Fila y Colas de Tareas
```sql
-- Bloqueo exclusivo inmediato (falla rápido si está ocupado)
SELECT * FROM cuentas WHERE id = 10 FOR UPDATE NOWAIT;

-- Consumir de cola omitiendo tareas bloqueadas por otros workers
SELECT * FROM cola_tareas WHERE estado = 'pendiente' 
ORDER BY creado_at ASC LIMIT 1 FOR UPDATE SKIP LOCKED;
```

### 3. Window Functions Clave
```sql
-- Cálculo acumulado en ventana temporal
SELECT fecha, monto, 
       SUM(monto) OVER (ORDER BY fecha ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS acumulado
FROM transacciones;

-- Comparativa con la fila anterior y posterior
SELECT mes, ventas,
       LAG(ventas, 1) OVER (ORDER BY mes) AS mes_anterior,
       LEAD(ventas, 1) OVER (ORDER BY mes) AS mes_siguiente
FROM ventas_mensuales;
```

### 4. Consultas Jerárquicas Recursivas
```sql
WITH RECURSIVE subordinados AS (
  SELECT id, nombre, jefe_id, 1 AS nivel
  FROM empleados WHERE id = $1 -- Miembro Ancla
  UNION ALL
  SELECT e.id, e.nombre, e.jefe_id, s.nivel + 1
  FROM empleados e
  JOIN subordinados s ON e.jefe_id = s.id -- Miembro Recursivo
)
SELECT * FROM subordinados;
```

### 5. Configurar Row-Level Security (RLS)
```sql
-- Habilitar RLS en una tabla
ALTER TABLE facturas ENABLE ROW LEVEL SECURITY;

-- Crear política de aislamiento dinámico por variable de sesión
CREATE POLICY policy_tenant_isolation ON facturas
    FOR ALL
    USING (tenant_id = NULLIF(current_setting('app.current_tenant_id', true), '')::integer);
```

---

> [!NOTE]
> ### 🧭 El Mapa de Desafíos y la Brújula del Programador
> 
> Afrontemos esta sección final de ejercicios mediante una analogía didáctica clara:
> 
> - **Los Desafíos (La Travesía por la Selva de Datos)**:
>   - Imagina que eres un explorador clásico que se adentra en una densa selva tropical en busca de un tesoro arqueológico (la información analítica oculta entre millones de filas caóticas).
>   - Resolver cada consulta de esta sección representa superar un obstáculo físico real en la selva:
>     - **El Nivel Fácil (Cruzar el riachuelo por el puente de madera)**: Requiere seguir las reglas básicas de equilibrio y filtros relacionales ordinarios.
>     - **El Nivel Medio (Escalar el muro de roca con cuerdas)**: Te obliga a coordinar fuerzas uniendo tablas con window functions y controlando la visibilidad del retrovisor.
>     - **El Nivel Experto (Cruzar el cañón del abismo oscilante en plena tormenta)**: Debes usar las técnicas más avanzadas de recursión, bloqueos en caliente concurrentes y aislamiento absoluto bajo RLS.
>   - **La Brújula (La Sintaxis SQL)**: El código limpio y estructurado es tu brújula magnética. Si entras en pánico y empiezas a dar vueltas en círculos escribiendo bucles iterativos en Node.js, te perderás en la selva y la latencia te devorará. Mantén la brújula apuntando al norte del álgebra declarativa y el motor de base de datos te abrirá el camino en microsegundos de forma fluida.

---

## 15.2 Ejercicios Resueltos Paso a Paso (Fácil a Experto)

---

### Desafío 1 (Fácil): Filtro de Auditoría de Transacciones Anómalas
**Problema**: Tienes una tabla `logs_acceso` y necesitas extraer todos los accesos sospechosos que ocurrieron en una fecha específica, ordenados cronológicamente de forma descendente, donde el correo del usuario contenga la palabra `"admin"` y el estado de la conexión sea `"error"`.

#### Paso 1: Configurar Datos
```sql
CREATE TABLE logs_acceso (
    id SERIAL PRIMARY KEY,
    email VARCHAR(150) NOT NULL,
    estado_conexion VARCHAR(20) NOT NULL,
    ip VARCHAR(50) NOT NULL,
    creado_at TIMESTAMP NOT NULL
);

INSERT INTO logs_acceso (email, estado_conexion, ip, creado_at) VALUES
('usuario1@test.com', 'exito', '192.168.1.5', '2026-05-22 10:00:00'),
('admin_finanzas@banco.com', 'error', '200.5.10.82', '2026-05-22 10:05:00'),
('developer_admin@corp.net', 'error', '198.51.100.12', '2026-05-22 10:12:00'),
('admin@test.com', 'exito', '192.168.1.10', '2026-05-22 10:15:00');
```

#### Paso 2: La Solución SQL
```sql
SELECT id, email, ip, creado_at
FROM logs_acceso
WHERE creado_at::date = '2026-05-22'
  AND email LIKE '%admin%'
  AND estado_conexion = 'error'
ORDER BY creado_at DESC;
```
* **Explicación**: El operador `LIKE '%admin%'` realiza una búsqueda de patrón. El casteo `creado_at::date` extrae limpiamente la parte de fecha ignorando las horas para optimizar el filtro exacto.

---

### Desafío 2 (Medio): El Ranking de Ventas por Categoría (Window Functions)
**Problema**: Tienes una tabla de `ventas` y necesitas obtener un listado de todos los vendedores que muestre su nombre, su categoría de producto, su total de ventas mensual, y una columna adicional con su **Ranking** de posición dentro de su categoría específica (donde el vendedor con mayores ventas de esa categoría sea el número 1).

#### Paso 1: Configurar Datos
```sql
CREATE TABLE ventas_mensuales_vendedores (
    id SERIAL PRIMARY KEY,
    vendedor VARCHAR(100) NOT NULL,
    categoria VARCHAR(50) NOT NULL,
    monto NUMERIC(12, 2) NOT NULL
);

INSERT INTO ventas_mensuales_vendedores (vendedor, categoria, monto) VALUES
('Sofia', 'Tecnología', 15000.00),
('Mateo', 'Tecnología', 22000.00),
('Valeria', 'Moda', 8500.00),
('Alejandro', 'Moda', 12000.00),
('Lucas', 'Tecnología', 18000.00),
('Catalina', 'Moda', 12000.00);
```

#### Paso 2: La Solución SQL
```sql
SELECT 
    vendedor,
    categoria,
    monto,
    -- DENSE_RANK no deja huecos en la clasificación en caso de empates exactos
    DENSE_RANK() OVER (
        PARTITION BY categoria 
        ORDER BY monto DESC
    ) AS ranking_en_categoria
FROM ventas_mensuales_vendedores
ORDER BY categoria, ranking_en_categoria;
```

#### Paso 3: Salida Esperada
| vendedor | categoria | monto | ranking_en_categoria |
| :--- | :--- | :--- | :--- |
| Alejandro | Moda | 12000.00 | 1 |
| Catalina | Moda | 12000.00 | 1 | (Empate)
| Valeria | Moda | 8500.00 | 2 |
| Mateo | Tecnología | 22000.00 | 1 |
| Lucas | Tecnología | 18000.00 | 2 |
| Sofia | Tecnología | 15000.00 | 3 |

* **Explicación**: `DENSE_RANK()` clasifica los registros de forma secuencial. Al usar `PARTITION BY categoria`, el ranking vuelve a empezar en 1 para cada grupo. `ORDER BY monto DESC` garantiza que las mayores ventas ocupen el puesto de honor en el podio analítico.

---

### Desafío 3 (Experto): Descomponer un Árbol Jerárquico Organizacional Inverso
**Problema**: Tienes un organigrama corporativo y un empleado específico te solicita una auditoría. Necesitas obtener la ruta jerárquica **hacia arriba** (es decir, desde el empleado seleccionado subiendo por sus jefes, gerentes y directores secuencialmente hasta llegar al CEO final), indicando el nivel de distancia y la ruta visual.

#### Paso 1: Configurar Datos
```sql
CREATE TABLE empleados_corporativo (
    id INT PRIMARY KEY,
    nombre VARCHAR(100) NOT NULL,
    jefe_id INT REFERENCES empleados_corporativo(id)
);

INSERT INTO empleados_corporativo (id, nombre, jefe_id) VALUES
(1, 'CEO - Alejandro Torres', NULL),
(2, 'VP Operaciones - Sofia Mendez', 1),
(3, 'Director Finanzas - Mateo Silva', 1),
(4, 'Gerente Logística - Valeria Rojas', 2),
(5, 'Coordinador Inventario - Lucas Gomez', 4),
(6, 'Analista Almacén - Catalina Rivas', 5);
```

#### Paso 2: La Solución SQL (CTE Recursiva Inversa)
```sql
WITH RECURSIVE ruta_ascendente AS (
  -- 1. Miembro Ancla: El empleado consultado (Catalina Rivas, ID = 6)
  SELECT 
    id, 
    nombre, 
    jefe_id, 
    1 AS nivel,
    nombre::text AS ruta_organizacional
  FROM empleados_corporativo
  WHERE id = 6

  UNION ALL

  -- 2. Miembro Recursivo: Unimos al jefe inmediato de la fila previa
  SELECT 
    e.id, 
    e.nombre, 
    e.jefe_id, 
    ra.nivel + 1 AS nivel,
    (ra.ruta_organizacional || ' -> ' || e.nombre)::text AS ruta_organizacional
  FROM empleados_corporativo e
  JOIN ruta_ascendente ra ON ra.jefe_id = e.id -- Unión clave hacia arriba
)
SELECT id, nombre, jefe_id, nivel, ruta_organizacional
FROM ruta_ascendente
ORDER BY nivel ASC;
```

#### Paso 3: Salida Esperada
| id | nombre | jefe_id | nivel | ruta_organizacional |
| :--- | :--- | :--- | :--- | :--- |
| 6 | Analista Almacén - Catalina Rivas | 5 | 1 | Catalina Rivas |
| 5 | Coordinador Inventario - Lucas Gomez | 4 | 2 | Catalina Rivas -> Lucas Gomez |
| 4 | Gerente Logística - Valeria Rojas | 2 | 3 | Catalina Rivas -> ... -> Valeria Rojas |
| 2 | VP Operaciones - Sofia Mendez | 1 | 4 | Catalina Rivas -> ... -> Sofia Mendez |
| 1 | CEO - Alejandro Torres | NULL | 5 | Catalina Rivas -> ... -> Alejandro Torres |

* **Explicación**: A diferencia de las recursiones habituales que bajan del CEO a los empleados, este desafío utiliza una recursión inversa apuntando la condición de unión `ra.jefe_id = e.id`. Esto obliga al motor a escalar recursivamente hacia arriba a lo largo del árbol hasta toparse con el CEO cuyo `jefe_id` es `NULL` (condición de parada natural de la recursión física).

---

## Resumen del Libro

¡Has concluido exitosamente el estudio de este volumen técnico de alto rendimiento relacional! A lo largo de este libro:
1. Deconstruiste los internals físicos del motor relacional PostgreSQL.
2. Indexaste con máxima selectividad usando B-Trees, Hash e Índices GIN en caliente.
3. Aseguraste transacciones atómicas e integras basadas en Write-Ahead Logging (WAL) y ACID.
4. Preveniste anomalías del mundo real como Write Skew utilizando Serializable e internals MVCC.
5. Dominaste la contención física mediante bloqueos controlados con `NOWAIT` y colas concurrentes `SKIP LOCKED`.
6. Resolviste análisis y jerarquías masivas con CTEs recursivas y Window Functions.
7. Modelaste bases de datos flexibles validadas con esquemas estrictos de JSONB de alto rendimiento.
8. Diagnosticaste cuellos de botella mediante la interpretación experta de `EXPLAIN ANALYZE`.
9. Escalaste el disco mediante estrategias de Particionamiento nativo y Sharding.
10. Protegiste los datos corporativos integrando arquitecturas seguras Multi-Tenant con Row-Level Security (RLS).

Estás plenamente capacitado para afrontar cualquier reto de ingeniería y base de datos relacional en producción de nivel empresarial. ¡Aplica este conocimiento y sigue construyendo sistemas de software legendarios!

---

[← Capítulo anterior (Capítulo 14)](14-proyecto-practico-saas.md) | [Inicio (README.md)](README.md)
