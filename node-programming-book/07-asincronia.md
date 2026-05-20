# Capítulo 7: Asincronía y el Event Loop

La asincronia es el corazon de Node.js. Entender el Event Loop, las Promesas y async/await es fundamental para escribir aplicaciones Node.js eficientes y correctas.

---

## 7.1 El Event Loop

El Event Loop es el mecanismo que permite a Node.js manejar miles de operaciones concurrentes en un solo hilo, delegando el I/O al sistema operativo.

### Fases del Event Loop

```
   ┌───────────────────────────┐
┌─>│           timers          │  setTimeout, setInterval
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐
│  │     pending callbacks     │  I/O callbacks diferidos
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐
│  │       idle, prepare       │  Interno
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐
│  │           poll            │  Esperar nuevos I/O, ejecutar I/O callbacks
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐
│  │           check           │  setImmediate
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐
│  │      close callbacks      │  socket.on("close", ...)
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐
└──┤   microtasks (cada fase)  │  process.nextTick, Promises
   └───────────────────────────┘
```

```typescript
// Demostracion del orden de ejecucion
console.log("1. Sincrono");

setTimeout(() => console.log("2. setTimeout (timers phase)"), 0);
setImmediate(() => console.log("3. setImmediate (check phase)"));

process.nextTick(() => console.log("4. nextTick (microtask)"));
Promise.resolve().then(() => console.log("5. Promise (microtask)"));

console.log("6. Sincrono final");

// Salida (Node.js):
// 1. Sincrono
// 6. Sincrono final
// 4. nextTick (microtask)    <- nextTick ANTES que Promises
// 5. Promise (microtask)
// 2. setTimeout (timers phase) o 3. setImmediate (orden no deterministico)
// 3. setImmediate (check phase) o 2. setTimeout
```

### Microtasks vs Macrotasks

```typescript
// Microtasks: process.nextTick, Promise.then/catch/finally, queueMicrotask
// Se ejecutan ENTRE cada fase del event loop, NUNCA dentro de una fase

// Macrotasks: setTimeout, setInterval, setImmediate, I/O callbacks
// Se ejecutan en fases especificas del event loop

// DEMOSTRACION: microtasks pueden bloquear el event loop
function bloquearConMicrotasks(): void {
  let contador = 0;

  function microtaskRecursivo(): void {
    contador++;
    if (contador < 100000) {
      Promise.resolve().then(microtaskRecursivo);
    }
  }

  microtaskRecursivo();

  setTimeout(() => console.log("NUNCA se ejecuta"), 0);
  // Las microtasks se ejecutan antes de pasar a la siguiente fase
  // El event loop jamas llega a la fase de timers
}
```

---

## 7.2 Timers (setTimeout, setInterval, setImmediate)

```typescript
// setTimeout: ejecutar una vez despues de un delay
const timeoutID: NodeJS.Timeout = setTimeout(() => {
  console.log("Ejecutado despues de 1 segundo");
}, 1000);

clearTimeout(timeoutID); // Cancelar

// setInterval: ejecutar periodicamente
const intervalID: NodeJS.Timeout = setInterval(() => {
  console.log("Cada 2 segundos");
}, 2000);

clearInterval(intervalID); // Detener

// setImmediate: ejecutar en la fase "check" del event loop
setImmediate(() => {
  console.log("Inmediato (despues de I/O)");
});

// setTimeout(fn, 0) vs setImmediate(fn)
// El orden es NO deterministico cuando se llaman desde el modulo principal
// DENTRO de un callback I/O, setImmediate SIEMPRE se ejecuta primero
import { readFile } from "node:fs";

readFile(__filename, () => {
  setTimeout(() => console.log("timeout"), 0);
  setImmediate(() => console.log("immediate"));
  // Output deterministico dentro de I/O:
  // immediate
  // timeout
});
```

### Patron: delay con Promesa

```typescript
// Convertir setTimeout en Promesa
function delay(ms: number): Promise<void> {
  return new Promise((resolve) => setTimeout(resolve, ms));
}

async function ejemplo() {
  console.log("Inicio");
  await delay(1000);
  console.log("Despues de 1 segundo");
  await delay(500);
  console.log("Despues de 1.5 segundos");
}

// delay con AbortSignal para cancelacion
function delayCancelable(ms: number, signal?: AbortSignal): Promise<void> {
  return new Promise((resolve, reject) => {
    if (signal?.aborted) {
      return reject(new Error("Abortado"));
    }

    const onAbort = (): void => {
      clearTimeout(id);
      reject(new Error("Abortado"));
    };

    signal?.addEventListener("abort", onAbort, { once: true });

    const id = setTimeout(() => {
      signal?.removeEventListener("abort", onAbort);
      resolve();
    }, ms);
  });
}
```

---

## 7.3 Promesas

### Creacion y estados

```typescript
// Estados: pending, fulfilled (resolved), rejected
const promesa = new Promise<string>((resolve, reject) => {
  // Operacion asincrona
  setTimeout(() => {
    const exito = Math.random() > 0.5;
    if (exito) {
      resolve("Operacion exitosa");
    } else {
      reject(new Error("Operacion fallida"));
    }
  }, 1000);
});

promesa
  .then((resultado) => {
    console.log("Exito:", resultado);   // string
    return resultado.length;
  })
  .then((longitud) => {
    console.log("Longitud:", longitud); // number
  })
  .catch((error: Error) => {
    console.error("Error:", error.message);
  })
  .finally(() => {
    console.log("Siempre se ejecuta (limpieza)");
  });
```

### Composicion de promesas

```typescript
// Promise.all: espera TODAS, falla si UNA falla
async function cargarMultiples() {
  const [usuarios, productos, config] = await Promise.all([
    fetch("/api/usuarios").then((r) => r.json()),
    fetch("/api/productos").then((r) => r.json()),
    fetch("/api/config").then((r) => r.json()),
  ]);
  console.log({ usuarios, productos, config });
}

// Promise.allSettled: espera TODAS, resultado de cada una
async function cargarConFallos() {
  const resultados = await Promise.allSettled([
    fetch("/api/existe").then((r) => r.json()),
    fetch("/api/no-existe").then((r) => r.json()),
    fetch("/api/tambien-existe").then((r) => r.json()),
  ]);

  for (const r of resultados) {
    if (r.status === "fulfilled") {
      console.log("OK:", r.value);
    } else {
      console.error("Error:", r.reason);
    }
  }
}

// Promise.race: primera en completar (exitosa o fallida)
async function conTimeout<T>(promesa: Promise<T>, ms: number): Promise<T> {
  const timeout = new Promise<never>((_, reject) =>
    setTimeout(() => reject(new Error("Timeout")), ms),
  );
  return Promise.race([promesa, timeout]);
}

// Promise.any: primera en completar EXITOSAMENTE
async function primerServicioDisponible() {
  try {
    const resultado = await Promise.any([
      fetch("https://api-primario.com/health"),
      fetch("https://api-secundario.com/health"),
      fetch("https://api-terciario.com/health"),
    ]);
    console.log("Servicio disponible:", resultado.url);
  } catch {
    console.log("Ningun servicio disponible");
  }
}
```

### Errores comunes con promesas

```typescript
// ERROR 1: Olvidar retornar la promesa en .then()
async function incorrecto() {
  obtenerDatos()
    .then((datos) => {
      procesarDatos(datos); // ❌ No se retorna, el await no espera
    });
}

async function correcto() {
  const datos = await obtenerDatos();
  await procesarDatos(datos);
}

// ERROR 2: No manejar rechazo de promesa
// Node.js muestra warning: UnhandledPromiseRejection
async function peligroso() {
  const promesa = fetch("/api"); // Si falla, no hay .catch()
  // Node.js warning + eventual crash en versiones futuras
}

// ERROR 3: async en funciones que no necesitan await
async function innecesario(): Promise<number> {  // ❌ async innecesario
  return 42;  // Equivalente a Promise.resolve(42)
}

function simple(): number {  // ✅ Mejor
  return 42;
}
```

---

## 7.4 Async/Await

### Sintaxis y patrones

```typescript
// Async function siempre retorna Promise<T>
async function obtenerUsuario(id: number): Promise<Usuario> {
  const respuesta = await fetch(`/api/usuarios/${id}`);
  if (!respuesta.ok) {
    throw new Error(`HTTP ${respuesta.status}: ${respuesta.statusText}`);
  }
  return respuesta.json() as Promise<Usuario>;
}

// try/catch con async/await
async function procesarUsuario(id: number): Promise<void> {
  try {
    const usuario = await obtenerUsuario(id);
    console.log("Usuario:", usuario);

    const permisos = await obtenerPermisos(usuario.rol);
    console.log("Permisos:", permisos);
  } catch (error) {
    if (error instanceof Error) {
      console.error("Error:", error.message);
    }
    throw error; // Re-lanzar si es necesario
  }
}

// Ejecutar en paralelo (no secuencial)
async function cargarDatos(userId: number): Promise<void> {
  // MAL: secuencial (una espera a la otra)
  const usuario = await obtenerUsuario(userId);
  const pedidos = await obtenerPedidos(userId);

  // BIEN: paralelo (ambas se inician al mismo tiempo)
  const [usuario2, pedidos2] = await Promise.all([
    obtenerUsuario(userId),
    obtenerPedidos(userId),
  ]);
}

// Patron: retry con async/await
async function conReintentos<T>(
  fn: () => Promise<T>,
  intentos: number,
  esperaMs: number,
): Promise<T> {
  for (let i = 0; i < intentos; i++) {
    try {
      return await fn();
    } catch (error) {
      if (i === intentos - 1) throw error;
      console.log(`Intento ${i + 1} fallido, reintentando...`);
      await delay(esperaMs * Math.pow(2, i));
    }
  }
  throw new Error("Inalcanzable");
}
```

### Top-level await (ES2022)

```typescript
// En modulos ESM (type: "module" en package.json)
// await se puede usar en el nivel superior del modulo

// src/index.ts (type: "module")
const config = await fetch("/api/config").then((r) => r.json());
const db = await crearConexion(config.database);

export { db, config };
// El modulo espera a que estas promesas se resuelvan antes de exportar
```

---

## 7.5 Workers Threads

Para tareas CPU-intensivas que bloquearian el event loop:

```typescript
// worker.ts
import { parentPort } from "node:worker_threads";

parentPort?.on("message", (datos: { numeros: number[] }) => {
  // Operacion CPU-intensiva
  const resultado = datos.numeros.map((n) => fibonacci(n));
  parentPort?.postMessage(resultado);
});

function fibonacci(n: number): number {
  if (n <= 1) return n;
  return fibonacci(n - 1) + fibonacci(n - 2);
}
```

```typescript
// main.ts
import { Worker } from "node:worker_threads";
import { cpus } from "node:os";

function ejecutarWorker(datos: { numeros: number[] }): Promise<number[]> {
  return new Promise((resolve, reject) => {
    const worker = new Worker("./worker.ts", {
      workerData: datos,
    });

    worker.on("message", resolve);
    worker.on("error", reject);
    worker.on("exit", (code) => {
      if (code !== 0) {
        reject(new Error(`Worker finalizo con codigo ${code}`));
      }
    });
  });
}

// Pool de workers
async function procesarEnParalelo(tareas: number[][]): Promise<number[][]> {
  const numWorkers = cpus().length;
  const resultados: Promise<number[]>[] = [];

  for (const tarea of tareas) {
    resultados.push(ejecutarWorker({ numeros: tarea }));
  }

  return Promise.all(resultados);
}
```

---

## 7.6 Streams de Node.js

```typescript
import { createReadStream, createWriteStream } from "node:fs";
import { pipeline } from "node:stream/promises";
import { Transform } from "node:stream";

// Leer archivo grande linea por linea sin cargar todo en memoria
async function procesarArchivoGrande(ruta: string): Promise<void> {
  const lineas = createReadStream(ruta, { encoding: "utf-8" });
  let contador = 0;

  for await (const chunk of lineas) {
    contador++;
    // Procesar cada chunk sin cargar todo el archivo
  }

  console.log(`Procesadas ${contador} lineas`);
}

// Transform stream: modificar datos en transito
const mayusculizador = new Transform({
  transform(chunk: Buffer, encoding, callback) {
    this.push(chunk.toString().toUpperCase());
    callback();
  },
});

// Pipeline con streams
async function copiarYTransformar(
  origen: string,
  destino: string,
): Promise<void> {
  await pipeline(
    createReadStream(origen),
    mayusculizador,
    createWriteStream(destino),
  );
  console.log("Archivo copiado y transformado");
}
```

---

## 7.7 AbortController: Cancelar Operaciones Asincronas

`AbortController` es el mecanismo estandar para cancelar cualquier operacion asincrona: fetch, timers, streams, event listeners y mas.

```typescript
// AbortController basico
const controller = new AbortController();
const signal = controller.signal;

// Cancelar fetch
async function fetchConTimeout(url: string, timeoutMs: number) {
  const controller = new AbortController();
  const timeoutId = setTimeout(() => controller.abort(), timeoutMs);

  try {
    const response = await fetch(url, { signal: controller.signal });
    clearTimeout(timeoutId);
    return response.json();
  } catch (error) {
    if (error instanceof DOMException && error.name === "AbortError") {
      console.log("Peticion cancelada");
      return null;
    }
    throw error;
  }
}

// Cancelar un timer con AbortSignal
function delayCancelable(ms: number, signal?: AbortSignal): Promise<void> {
  return new Promise((resolve, reject) => {
    if (signal?.aborted) {
      return reject(signal.reason ?? new Error("Abortado"));
    }

    const onAbort = () => {
      clearTimeout(id);
      reject(signal!.reason ?? new Error("Abortado"));
    };

    signal?.addEventListener("abort", onAbort, { once: true });

    const id = setTimeout(() => {
      signal?.removeEventListener("abort", onAbort);
      resolve();
    }, ms);
  });
}

// Cancelar multiples operaciones
async function cargarConCancelacion(urls: string[], signal: AbortSignal) {
  const resultados = await Promise.all(
    urls.map((url) => fetch(url, { signal }).then((r) => r.json())),
  );
  return resultados;
}

// Usar AbortSignal.timeout (Node 17+, ES2024)
async function fetchRapido(url: string) {
  const response = await fetch(url, {
    signal: AbortSignal.timeout(3000), // Timeout de 3 segundos
  });
  return response.json();
}

// any(signals): cancela cuando cualquiera aborta
const signal = AbortSignal.any([
  AbortSignal.timeout(5000),
  controller.signal,
]);
```

---

## 7.8 AsyncLocalStorage: Contexto de Peticion

`AsyncLocalStorage` permite propagar contexto (request ID, usuario, tenant) a traves de toda la cadena asincrona sin pasarlo como parametro.

```typescript
import { AsyncLocalStorage } from "node:async_hooks";

interface RequestContext {
  requestId: string;
  usuarioId?: string;
  tenant?: string;
}

const als = new AsyncLocalStorage<RequestContext>();

// Middleware que inicia el contexto
function contextoMiddleware(req: Request, res: Response, next: NextFunction) {
  const ctx: RequestContext = {
    requestId: crypto.randomUUID(),
    usuarioId: req.headers["x-user-id"] as string,
  };

  als.run(ctx, () => {
    // Todo el codigo asincrono dentro de run() hereda el contexto
    next();
  });
}

// Obtener contexto desde cualquier lugar
function getContext(): RequestContext | undefined {
  return als.getStore();
}

// Uso en servicios (sin pasar contexto como parametro)
class UsuarioService {
  async obtenerPerfil() {
    const ctx = getContext();
    logger.info("Obteniendo perfil", {
      requestId: ctx?.requestId,
      usuarioId: ctx?.usuarioId,
    });
    // ...
  }
}

// Logger con contexto automatico
import pino from "pino";

const logger = pino({
  mixin() {
    const ctx = getContext();
    return ctx ? { requestId: ctx.requestId } : {};
  },
});
```

---

## 7.9 Promise.withResolvers y queueMicrotask

```typescript
// Promise.withResolvers (ES2024 / Node 22+): crea Promise + resolve/reject
const { promise, resolve, reject } = Promise.withResolvers<string>();

// Pasar resolve/reject a otro contexto
setTimeout(() => resolve("hecho"), 1000);
const resultado = await promise;
console.log(resultado); // "hecho"

// Sin withResolvers (antes):
function crearPromiseConResolvers<T>() {
  let resolve!: (value: T) => void;
  let reject!: (reason?: unknown) => void;
  const promise = new Promise<T>((res, rej) => {
    resolve = res;
    reject = rej;
  });
  return { promise, resolve, reject };
}

// queueMicrotask: programar microtask
console.log("1");
queueMicrotask(() => console.log("2 (microtask)"));
Promise.resolve().then(() => console.log("3 (microtask)"));
console.log("4");
// Output: 1, 4, 2, 3
```

---

## 7.10 Atomics y SharedArrayBuffer (Workers)

```typescript
// worker.ts (Worker Thread)
import { parentPort } from "node:worker_threads";

parentPort?.on("message", (sharedBuffer: SharedArrayBuffer) => {
  const vista = new Int32Array(sharedBuffer);

  // Operacion atomica: incrementar
  Atomics.add(vista, 0, 1);

  // Notificar al hilo principal
  Atomics.notify(vista, 0, 1);

  parentPort?.postMessage("incrementado");
});
```

```typescript
// main.ts
import { Worker } from "node:worker_threads";

const sharedBuffer = new SharedArrayBuffer(4); // 4 bytes = 1 Int32
const vista = new Int32Array(sharedBuffer);

const worker = new Worker("./worker.ts");
worker.postMessage(sharedBuffer);

// Esperar notificacion atomica (sin busy-waiting)
Atomics.wait(vista, 0, 0); // Espera hasta que cambie
console.log("Valor final:", vista[0]); // 1
```

---

## 7.11 Concurrencia Estructurada y Backpressure

### Limitar concurrencia con p-limit

```typescript
import pLimit from "p-limit";

// ❌ Promise.all sin limite: 10000 requests simultaneos = 💀
const resultados = await Promise.all(
  urls.map((url) => fetch(url)),
);

// ✅ Limitar a 5 requests concurrentes
const limit = pLimit(5);
const resultados = await Promise.all(
  urls.map((url) => limit(() => fetch(url))),
);

// Patron: procesar lote con concurrencia controlada
async function procesarLote<T, R>(
  items: T[],
  concurrency: number,
  fn: (item: T) => Promise<R>,
): Promise<R[]> {
  const limit = pLimit(concurrency);
  return Promise.all(items.map((item) => limit(() => fn(item))));
}
```

### fastq: cola de trabajo con control de flujo

```typescript
import fastq from "fastq";
import type { queueAsPromised } from "fastq";

const worker = async (task: { id: number; url: string }): Promise<void> => {
  const response = await fetch(task.url);
  console.log(`Task ${task.id}: ${response.status}`);
};

const cola: queueAsPromised<{ id: number; url: string }> = fastq.promise(worker, 3);

// Encolar trabajos (se procesan de a 3)
for (let i = 0; i < 100; i++) {
  await cola.push({ id: i, url: `https://api.ejemplo.com/item/${i}` });
}

// La cola aplica backpressure: push espera si el buffer esta lleno
console.log("Todos los trabajos completados");
```

### Backpressure en streams personalizados

```typescript
// Implementar backpressure en un Readable stream
class GeneradorDatos extends Readable {
  private maxEnVuelo = 10;
  private enVuelo = 0;

  constructor(private fuente: AsyncIterator<unknown>) {
    super({ objectMode: true, highWaterMark: 5 }); // Buffer de 5
  }

  async _read(): Promise<void> {
    // Respetar backpressure: no leer mas si el buffer esta lleno
    if (this.enVuelo >= this.maxEnVuelo) return;

    this.enVuelo++;
    try {
      const { done, value } = await this.fuente.next();
      if (done) {
        this.push(null);
      } else {
        const puedeSeguir = this.push(value);
        if (!puedeSeguir) {
          // Consumidor lento: parar de producir
          console.log("Backpressure: pausando produccion");
        }
      }
    } finally {
      this.enVuelo--;
    }
  }
}
```

---

## 7.12 Load Shedding: Rechazar Temprano

```typescript
// Load shedding: rechazar peticiones cuando el sistema esta sobrecargado
// En vez de aceptarlas y que todas fallen lentamente (timeout cascade)

class LoadShedder {
  private eventLoopLag = 0;
  private maxLag = 100; // ms

  constructor() {
    // Monitorear event loop lag
    const hist = monitorEventLoopDelay();
    hist.enable();
    setInterval(() => {
      this.eventLoopLag = hist.mean / 1e6;
    }, 1000);
  }

  shouldShed(): boolean {
    const cpuUsage = process.cpuUsage();
    const memoryUsage = process.memoryUsage().heapUsed / process.memoryUsage().heapTotal;

    // Rechazar si:
    // 1. Event loop lag > 100ms
    // 2. Heap > 90%
    if (this.eventLoopLag > this.maxLag) return true;
    if (memoryUsage > 0.9) return true;

    return false;
  }
}

// NestJS: load shedding middleware
@Injectable()
export class LoadSheddingGuard implements CanActivate {
  constructor(private shedder: LoadShedder) {}

  canActivate(): boolean {
    if (this.shedder.shouldShed()) {
      throw new HttpException("Servicio sobrecargado", 503);
    }
    return true;
  }
}

// Aplicar globalmente
app.useGlobalGuards(new LoadSheddingGuard());
```

---

## Resumen del Capítulo

- El Event Loop tiene 6 fases. Microtasks (`nextTick`, Promises) se ejecutan entre fases.
- `setTimeout` es para ejecucion retardada unica; `setInterval` para periodica; `setImmediate` para ejecutar en la fase check.
- Las promesas tienen 3 estados: pending, fulfilled, rejected. Encadenar con `.then().catch().finally()`.
- `Promise.all` espera todas (falla si una falla). `allSettled` espera todas (resultado individual). `race` primera en completar. `any` primera exitosa.
- `async/await` es azucar sintactico sobre promesas. Codigo asincrono que se lee como sincrono.
- **Nunca** bloquees el event loop. Tareas CPU-intensivas van en Worker Threads.
- Streams permiten procesar grandes volumenes de datos sin cargarlos en memoria.
- Top-level await permite usar `await` en el nivel superior de modulos ESM.

En el siguiente capitulo exploraremos el manejo de errores en profundidad.

---

← [Capítulo anterior](06-interfaces.md) | [Inicio](README.md) | [Capítulo siguiente →](08-manejo-de-errores.md)
