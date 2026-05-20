# Capítulo 8: Manejo de Errores

Un manejo de errores robusto es lo que separa una aplicacion de juguete de una lista para produccion. Node.js ofrece multiples mecanismos: excepciones sincronas, rechazo de promesas, errores en streams y eventos de error.

> "No lances errores que no vas a manejar. No captures errores que no sabes como manejar."

---

## 8.1 try/catch/finally en Profundidad

```typescript
// try/catch clasico
function parsearJSON(json: string): unknown {
  try {
    return JSON.parse(json);
  } catch (error) {
    // Type narrowing: error es unknown en catch
    if (error instanceof SyntaxError) {
      console.error("JSON invalido:", error.message);
    }
    return null;
  }
}

// finally: siempre se ejecuta (con o sin error)
function leerArchivoSeguro(ruta: string): string | null {
  let descriptor: number | undefined;

  try {
    descriptor = fs.openSync(ruta, "r");
    const contenido = fs.readFileSync(ruta, "utf-8");
    return contenido;
  } catch (error) {
    console.error("Error al leer archivo:", error);
    return null;
  } finally {
    if (descriptor !== undefined) {
      fs.closeSync(descriptor);
    }
  }
}
```

### Multiple catch (error type narrowing)

```typescript
async function operacionRiesgosa(): Promise<void> {
  try {
    await fetch("/api/datos");
  } catch (error) {
    if (error instanceof TypeError) {
      // Error de red (fetch fallo)
      console.error("Error de red:", error.message);
    } else if (error instanceof SyntaxError) {
      // Error al parsear JSON
      console.error("JSON invalido:", error.message);
    } else if (error instanceof Error) {
      // Error generico con stack
      console.error("Error:", error.message, error.stack);
    } else {
      // Podria ser cualquier cosa lanzada (string, number, etc.)
      console.error("Error desconocido:", error);
    }
  }
}
```

---

## 8.2 Errores Personalizados

```typescript
// Clase base para errores de aplicacion
class AppError extends Error {
  constructor(
    message: string,
    public readonly codigo: string,
    public readonly httpStatus: number = 500,
    public readonly causa?: Error,
  ) {
    super(message);
    this.name = "AppError";
    // Corregir el prototype para instanceof
    Object.setPrototypeOf(this, AppError.prototype);
  }

  toJSON(): Record<string, unknown> {
    return {
      codigo: this.codigo,
      mensaje: this.message,
      httpStatus: this.httpStatus,
      ...(this.causa && { causa: this.causa.message }),
    };
  }
}

// Errores de dominio especificos
class NotFoundError extends AppError {
  constructor(recurso: string, id: string) {
    super(
      `${recurso} con id '${id}' no encontrado`,
      "NOT_FOUND",
      404,
    );
    this.name = "NotFoundError";
    Object.setPrototypeOf(this, NotFoundError.prototype);
  }
}

class ValidationError extends AppError {
  constructor(
    message: string,
    public readonly errores: Array<{ campo: string; mensaje: string }>,
  ) {
    super(message, "VALIDATION_ERROR", 400);
    this.name = "ValidationError";
    Object.setPrototypeOf(this, ValidationError.prototype);
  }

  override toJSON(): Record<string, unknown> {
    return {
      ...super.toJSON(),
      errores: this.errores,
    };
  }
}

class DatabaseError extends AppError {
  constructor(message: string, causa?: Error) {
    super(message, "DATABASE_ERROR", 500, causa);
    this.name = "DatabaseError";
    Object.setPrototypeOf(this, DatabaseError.prototype);
  }
}
```

### Uso de errores personalizados

```typescript
async function obtenerUsuario(id: string): Promise<Usuario> {
  const usuario = await db.usuario.findUnique({ where: { id } });

  if (!usuario) {
    throw new NotFoundError("Usuario", id);
  }

  return usuario;
}

// En el handler HTTP
async function handler(req: Request, res: Response): Promise<void> {
  try {
    const usuario = await obtenerUsuario(req.params.id);
    res.json(usuario);
  } catch (error) {
    if (error instanceof NotFoundError) {
      res.status(404).json(error.toJSON());
    } else if (error instanceof AppError) {
      res.status(error.httpStatus).json(error.toJSON());
    } else {
      console.error("Error inesperado:", error);
      res.status(500).json({
        codigo: "INTERNAL_ERROR",
        mensaje: "Error interno del servidor",
      });
    }
  }
}
```

---

## 8.3 Errores en Promesas y Async/Await

### Patrones de manejo

```typescript
// Patron 1: try/catch con async/await
async function seguro(): Promise<void> {
  try {
    const datos = await fetch("/api").then((r) => r.json());
    console.log(datos);
  } catch (error) {
    console.error("Fallo al obtener datos:", error);
  }
}

// Patron 2: .catch() en la promesa
function conFallback(): Promise<Datos> {
  return fetch("/api/primario")
    .then((r) => r.json())
    .catch(() => fetch("/api/secundario"))
    .then((r) => r.json())
    .catch(() => datosPorDefecto);
}

// Patron 3: wrapper de seguridad (nunca lanza)
async function seguroNuncaLanza<T>(
  fn: () => Promise<T>,
  fallback: T,
): Promise<T> {
  try {
    return await fn();
  } catch {
    return fallback;
  }
}

// Patron 4: Result type (estilo Go/Rust)
type Result<T, E = Error> =
  | { ok: true; value: T }
  | { ok: false; error: E };

async function toResult<T>(promesa: Promise<T>): Promise<Result<T>> {
  try {
    const value = await promesa;
    return { ok: true, value };
  } catch (error) {
    return { ok: false, error: error as Error };
  }
}

// Uso del patron Result
async function ejemplo() {
  const resultado = await toResult(fetch("/api/usuarios").then((r) => r.json()));

  if (resultado.ok) {
    console.log("Usuarios:", resultado.value);
  } else {
    console.error("Error:", resultado.error.message);
  }
}
```

---

## 8.4 Manejo Global de Errores

### Uncaught exceptions y unhandled rejections

```typescript
// Errores sincronos no capturados
process.on("uncaughtException", (error: Error) => {
  console.error("Error no capturado:", error);
  // Loggear, notificar, hacer graceful shutdown
  process.exit(1); // Salir: el proceso esta en estado inconsistente
});

// Promesas rechazadas sin .catch()
process.on("unhandledRejection", (reason: unknown, promise: Promise<unknown>) => {
  console.error("Promesa rechazada sin manejar:", reason);
  // No hacer process.exit() aqui necesariamente
  // En Node.js moderno, esto termina el proceso eventualmente
});

// Advertencias
process.on("warning", (warning: Error) => {
  console.warn("Warning:", warning.name, warning.message);
});
```

### Error handler centralizado para servidores HTTP

```typescript
import { Request, Response, NextFunction } from "express";

// Middleware de error (Express)
function errorHandler(
  error: Error,
  req: Request,
  res: Response,
  next: NextFunction,
): void {
  if (error instanceof AppError) {
    res.status(error.httpStatus).json(error.toJSON());
    return;
  }

  // Error desconocido: loggear y responder generico
  console.error("Error inesperado:", {
    mensaje: error.message,
    stack: error.stack,
    ruta: req.path,
    metodo: req.method,
  });

  res.status(500).json({
    codigo: "INTERNAL_ERROR",
    mensaje: "Error interno del servidor",
  });
}
```

---

## 8.5 Errores en Streams y EventEmitter

```typescript
import { createReadStream, createWriteStream } from "node:fs";
import { EventEmitter } from "node:events";

// Error en streams: siempre manejar el evento 'error'
function copiarArchivo(origen: string, destino: string): Promise<void> {
  return new Promise((resolve, reject) => {
    const lectura = createReadStream(origen);
    const escritura = createWriteStream(destino);

    lectura.on("error", reject);
    escritura.on("error", reject);
    escritura.on("finish", resolve);

    lectura.pipe(escritura);
  });
}

// Patron: EventEmitter con manejo de errores tipado
interface MiEmitter {
  on(event: "error", listener: (error: Error) => void): this;
  on(event: "datos", listener: (datos: string) => void): this;
  on(event: "cerrar", listener: () => void): this;
}

// Regla de oro: si un EventEmitter puede emitir 'error',
// SIEMPRE agrega un listener para 'error'
```

---

## 8.6 Winston y Logging Estructurado

```typescript
import winston from "winston";

const logger = winston.createLogger({
  level: process.env.LOG_LEVEL ?? "info",
  format: winston.format.combine(
    winston.format.timestamp(),
    winston.format.errors({ stack: true }),
    winston.format.splat(),
    winston.format.json(),
  ),
  defaultMeta: { servicio: "api-usuarios" },
  transports: [
    // Errores a archivo separado
    new winston.transports.File({
      filename: "logs/error.log",
      level: "error",
      maxsize: 5 * 1024 * 1024, // 5MB
      maxFiles: 5,
    }),
    // Todos los logs a archivo
    new winston.transports.File({
      filename: "logs/combined.log",
      maxsize: 10 * 1024 * 1024,
      maxFiles: 10,
    }),
  ],
});

// En desarrollo: tambien loggear a consola
if (process.env.NODE_ENV !== "production") {
  logger.add(
    new winston.transports.Console({
      format: winston.format.combine(
        winston.format.colorize(),
        winston.format.simple(),
      ),
    }),
  );
}

// Uso
logger.info("Servidor iniciado", { puerto: 8080 });
logger.error("Fallo al conectar BD", {
  host: process.env.DB_HOST,
  error: error.message,
});

// Interceptar console.log para que use winston
const originalConsole = { ...console };
console.log = (...args) => logger.info(args.join(" "));
console.error = (...args) => logger.error(args.join(" "));
```

---

## 8.7 Buenas Practicas

### Checklist de manejo de errores

```typescript
// 1. SIEMPRE manejar errores de promesas
// ❌ MAL
fetch("/api").then((r) => r.json());

// ✅ BIEN
fetch("/api")
  .then((r) => r.json())
  .catch((error) => logger.error("Error fetching API", { error }));

// 2. NO lanzar strings o numeros
// ❌ MAL
throw "algo fallo";
throw 404;

// ✅ BIEN
throw new Error("algo fallo");
throw new NotFoundError("Usuario", id);

// 3. PROPORCIONAR contexto util en el mensaje
// ❌ MAL
throw new Error("Error");

// ✅ BIEN
throw new Error(`Fallo al procesar pago: orden=${ordenId}, monto=${monto}`);

// 4. DISTINGUIR errores operacionales vs de programacion
// Operacional: esperado, manejable (timeout, archivo no existe)
// Programacion: bug, no deberia ocurrir (undefined is not a function)

// 5. USAR error.cause para encadenar errores
try {
  await guardarEnBD(datos);
} catch (error) {
  throw new Error("Fallo al guardar usuario", { cause: error });
}
```

---

## 8.8 Error.cause y Encadenamiento

```typescript
// Error.cause: encadena errores preservando la causa raiz
async function guardarUsuario(datos: unknown) {
  try {
    await db.usuario.create({ data: datos });
  } catch (error) {
    throw new Error("Fallo al guardar usuario en BD", { cause: error });
  }
}

async function registrarUsuario(datos: { email: string; nombre: string }) {
  try {
    await guardarUsuario(datos);
  } catch (error) {
    throw new Error(`No se pudo registrar usuario ${datos.email}`, { cause: error });
  }
}

// Inspeccionar la cadena de causas
try {
  await registrarUsuario({ email: "test@test.com", nombre: "Andres" });
} catch (error) {
  if (error instanceof Error) {
    console.error(error.message);
    // "No se pudo registrar usuario test@test.com"
    console.error(error.cause);
    // Error: Fallo al guardar usuario en BD
    //   [cause]: PrismaClientKnownRequestError: ...
  }
}
```

---

## 8.9 AggregateError

```typescript
// AggregateError: agrupa multiples errores (Promise.any, operaciones en paralelo)

async function procesarArchivos(archivos: string[]) {
  const errores: Error[] = [];
  const resultados: string[] = [];

  for (const archivo of archivos) {
    try {
      const contenido = await fs.readFile(archivo, "utf-8");
      resultados.push(contenido);
    } catch (error) {
      errores.push(new Error(`Error al leer ${archivo}`, { cause: error }));
    }
  }

  if (errores.length > 0) {
    throw new AggregateError(errores, `${errores.length} archivos fallaron`);
  }
  return resultados;
}

try {
  await procesarArchivos(["existe.txt", "no-existe.txt", "otro-no-existe.txt"]);
} catch (error) {
  if (error instanceof AggregateError) {
    console.error(`${error.errors.length} errores:`);
    for (const e of error.errors) {
      console.error("  -", e.message);
    }
  }
}
```

---

## 8.10 Codigos de Error (Convencion Node.js)

```typescript
// Node.js usa codigos de error como strings (ENOENT, ECONNREFUSED, etc.)
import { readFile } from "node:fs/promises";

try {
  await readFile("/ruta/inexistente.txt");
} catch (error) {
  if (error instanceof Error && "code" in error) {
    switch (error.code) {
      case "ENOENT":
        console.error("Archivo no encontrado");
        break;
      case "EACCES":
        console.error("Permiso denegado");
        break;
      case "ECONNREFUSED":
        console.error("Conexion rechazada");
        break;
      default:
        console.error(`Error de sistema: ${error.code}`);
    }
  }
}

// Crear tus propios codigos de error
class AppError extends Error {
  constructor(
    message: string,
    public readonly code: string,    // ej: "VALIDATION_ERROR", "NOT_FOUND"
    public readonly statusCode: number = 500,
    options?: { cause?: Error },
  ) {
    super(message, options);
  }
}

throw new AppError("Email invalido", "VALIDATION_ERROR", 400);
```

---

## 8.11 Operational vs Programmer Errors

```typescript
// Errores OPERACIONALES: esperados en runtime, se manejan con gracia
// - Conexion de BD caida (ECONNREFUSED)
// - Archivo no encontrado (ENOENT)
// - Timeout de peticion HTTP
// - Validacion de entrada del usuario
// -> MANEJAR: retry, fallback, mensaje al usuario

// Errores DE PROGRAMACION: bugs, nunca deberian ocurrir
// - TypeError: undefined is not a function
// - ReferenceError: variable no declarada
// - Pasar argumentos incorrectos a una funcion
// - Olvidar await en una promesa
// -> NO MANEJAR: arreglar el bug, tests, typechecking

// Estrategia: distinguir en el handler global
process.on("uncaughtException", (error) => {
  if (error instanceof AppError) {
    logger.warn("Error operacional:", { code: error.code, message: error.message });
    // Avisar al equipo pero NO reiniciar necesariamente
  } else {
    logger.error("Error de programacion (BUG):", error);
    // SIEMPRE reiniciar: el proceso esta en estado inconsistente
    process.exit(1);
  }
});
```

---

## Resumen del Capítulo

- `try/catch/finally` para errores sincronos. `finally` siempre se ejecuta.
- Crea jerarquias de errores con `extends Error` y prototipo corregido.
- `error instanceof X` para type narrowing en catch.
- Patron Result (`{ ok: true; value: T } | { ok: false; error: E }`) como alternativa a throw.
- `process.on("uncaughtException")` y `"unhandledRejection"` para manejo global.
- Los streams emiten `"error"`: nunca olvides manejar ese evento.
- Winston proporciona logging estructurado con niveles, transportes y formatos.
- Proporciona contexto en los mensajes de error. Encadena errores con `{ cause: error }`.

En el siguiente capitulo exploraremos los modulos, paquetes y el ecosistema npm.
