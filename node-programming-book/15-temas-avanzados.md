# Capítulo 15: Temas Avanzados

Este capitulo cubre herramientas y tecnicas avanzadas para desarrolladores Node.js y TypeScript: decorators, metaprogramacion, Streams, profiling, PM2, seguridad y gRPC.

---

## 15.1 Decorators

### Decorators TC39 (Stage 3, futuro estandar)

```typescript
// Decorator de metodo: logging
function log(
  target: Function,
  context: ClassMethodDecoratorContext,
) {
  const methodName = String(context.name);

  function replacementMethod(this: any, ...args: any[]) {
    console.log(`LOG: Entrando a ${methodName}(${args.join(", ")})`);
    const result = target.call(this, ...args);
    console.log(`LOG: Saliendo de ${methodName} => ${result}`);
    return result;
  }

  return replacementMethod;
}

class Calculadora {
  @log
  sumar(a: number, b: number): number {
    return a + b;
  }
}

const calc = new Calculadora();
calc.sumar(3, 4);
// LOG: Entrando a sumar(3, 4)
// LOG: Saliendo de sumar => 7
```

### Decorators legacy (experimental, NestJS)

```typescript
// Decorator de clase
function Injectable(): ClassDecorator {
  return (target) => {
    Reflect.defineMetadata("injectable", true, target);
  };
}

// Decorator de parametro
function Inject(token: string): ParameterDecorator {
  return (target, propertyKey, parameterIndex) => {
    const existing = Reflect.getOwnMetadata("inject", target, propertyKey) || [];
    existing.push({ index: parameterIndex, token });
    Reflect.defineMetadata("inject", existing, target, propertyKey);
  };
}

@Injectable()
class UsuarioService {
  constructor(
    @Inject("UsuarioRepository") private repo: any,
  ) {}
}
```

---

## 15.2 Streams Avanzados

```typescript
import { Readable, Writable, Transform, pipeline } from "node:stream";
import { promisify } from "node:util";

const pipelineAsync = promisify(pipeline);

// Custom Readable: genera datos
class GeneradorNumeros extends Readable {
  private actual = 0;

  constructor(private max: number) {
    super({ objectMode: true });
  }

  _read(): void {
    if (this.actual >= this.max) {
      this.push(null); // Fin del stream
      return;
    }
    this.push(this.actual++);
  }
}

// Custom Transform: filtra y transforma
class FiltrarPares extends Transform {
  constructor() {
    super({ objectMode: true });
  }

  _transform(chunk: number, encoding: string, callback: Function): void {
    if (chunk % 2 === 0) {
      this.push(chunk * 2);
    }
    callback();
  }
}

// Custom Writable: acumula en array
class Colector<T> extends Writable {
  public readonly datos: T[] = [];

  constructor() {
    super({ objectMode: true });
  }

  _write(chunk: T, encoding: string, callback: Function): void {
    this.datos.push(chunk);
    callback();
  }
}

// Uso
const generador = new GeneradorNumeros(20);
const filtro = new FiltrarPares();
const colector = new Colector<number>();

await pipelineAsync(generador, filtro, colector);
console.log(colector.datos); // [0, 4, 8, 12, 16, 20, 24, 28, 32, 36]
```

### Streams con async iterators

```typescript
// Async generator: lee archivo linea por linea
import { createReadStream } from "node:fs";
import { createInterface } from "node:readline";

async function* leerLineas(ruta: string): AsyncGenerator<string> {
  const rl = createInterface({
    input: createReadStream(ruta),
    crlfDelay: Infinity,
  });

  for await (const linea of rl) {
    yield linea;
  }
}

// Procesar archivo grande sin memoria
async function procesarArchivo(ruta: string) {
  for await (const linea of leerLineas(ruta)) {
    await procesarLinea(linea);
  }
}
```

---

## 15.3 Performance y Profiling

### Medir con Performance Hooks

```typescript
import { performance, PerformanceObserver } from "node:perf_hooks";

const obs = new PerformanceObserver((items) => {
  for (const entry of items.getEntries()) {
    console.log(`${entry.name}: ${entry.duration.toFixed(2)}ms`);
  }
});
obs.observe({ type: "measure" });

performance.mark("inicio-operacion");
await operacionCostosa();
performance.mark("fin-operacion");
performance.measure("operacion-costosa", "inicio-operacion", "fin-operacion");
```

### CPU Profiling con Clinic.js

```bash
# Instalar
npm install -g clinic

# Perfil de CPU (flamegraph)
clinic doctor -- node dist/index.js

# Perfil de event loop
clinic bubbleprof -- node dist/index.js

# Perfil de memoria
clinic heapprofiler -- node dist/index.js

# Abrir resultados (interactivo)
clinic flame
```

### Detectar memory leaks

```typescript
import { heapSnapshot } from "node:v8";

// Tomar snapshot en momentos clave
const snapshot1 = heapSnapshot();
// ... ejecutar operacion sospechosa ...
const snapshot2 = heapSnapshot();

// Comparar con Chrome DevTools
// 1. node --inspect dist/index.js
// 2. Chrome: chrome://inspect
// 3. Memory tab -> Take snapshot -> Compare

// O con --heap-prof para perfil automatico
// node --heap-prof dist/index.js
```

---

## 15.4 Cluster y PM2

```typescript
// cluster.ts: usar todos los cores de CPU
import cluster from "node:cluster";
import { cpus } from "node:os";
import { createServer } from "node:http";

if (cluster.isPrimary) {
  const numCPUs = cpus().length;
  console.log(`Master ${process.pid} iniciando ${numCPUs} workers`);

  for (let i = 0; i < numCPUs; i++) {
    cluster.fork();
  }

  cluster.on("exit", (worker) => {
    console.log(`Worker ${worker.process.pid} murio. Reiniciando...`);
    cluster.fork();
  });
} else {
  createServer((req, res) => {
    res.end(`Worker ${process.pid}\n`);
  }).listen(8080);

  console.log(`Worker ${process.pid} iniciado`);
}
```

### PM2 (recomendado para produccion)

```bash
# Instalar
npm install -g pm2

# Iniciar con cluster mode
pm2 start dist/index.js -i max --name mi-api

# Ver estado
pm2 status
pm2 monit

# Logs
pm2 logs mi-api

# Reinicio sin downtime (graceful reload)
pm2 reload mi-api

# Guardar configuracion para reinicio automatico
pm2 save
pm2 startup

# Configuracion ecosystem.config.js
module.exports = {
  apps: [{
    name: "mi-api",
    script: "./dist/index.js",
    instances: "max",
    exec_mode: "cluster",
    env: { NODE_ENV: "production" },
    max_memory_restart: "500M",
    error_file: "/var/log/mi-api/error.log",
    out_file: "/var/log/mi-api/out.log",
  }],
};
```

---

## 15.5 Seguridad

```typescript
// Helmet: headers de seguridad HTTP
import helmet from "helmet";
app.use(helmet());

// Rate limiting
import rateLimit from "express-rate-limit";

const limiter = rateLimit({
  windowMs: 15 * 60 * 1000, // 15 minutos
  max: 100,                  // 100 peticiones por ventana
  standardHeaders: true,
  legacyHeaders: false,
});
app.use("/api", limiter);

// Validar entrada con Zod SIEMPRE
const schema = z.object({
  email: z.string().email().max(255),
  nombre: z.string().min(1).max(100).regex(/^[a-zA-Záéíóúü\s]+$/),
  edad: z.number().int().min(0).max(150),
});

// Sanitizar salida: nunca exponer errores internos
// ❌ res.status(500).json({ error: error.stack });
// ✅ res.status(500).json({ error: "Error interno del servidor" });
```

### Cheat sheet de seguridad

| Riesgo | Solucion |
|--------|----------|
| Inyeccion SQL | Prisma/Drizzle/Knex parametrizado |
| XSS | Helmet + sanitizar output |
| CSRF | Tokens CSRF o SameSite cookies |
| Rate limiting | express-rate-limit |
| Fuerza bruta | rate-limit por IP + login |
| Secretos expuestos | Variables de entorno, nunca hardcodeados |
| Dependencias vulnerables | `npm audit` / `pnpm audit` regularmente |
| Headers inseguros | Helmet (11 middlewares de seguridad) |
| Denial of Service | Timeouts, max body size, max connections |

---

## 15.6 gRPC y Protobuf

```protobuf
// proto/usuario.proto
syntax = "proto3";

package usuario;

service UsuarioService {
  rpc Crear(CrearUsuarioRequest) returns (UsuarioResponse);
  rpc ObtenerPorId(ObtenerPorIdRequest) returns (UsuarioResponse);
}

message CrearUsuarioRequest {
  string nombre = 1;
  string email = 2;
}

message ObtenerPorIdRequest {
  string id = 1;
}

message UsuarioResponse {
  string id = 1;
  string nombre = 2;
  string email = 3;
  string creado_en = 4;
}
```

```typescript
// server.ts
import * as grpc from "@grpc/grpc-js";
import * as protoLoader from "@grpc/proto-loader";

const packageDefinition = protoLoader.loadSync("proto/usuario.proto");
const usuarioProto = grpc.loadPackageDefinition(packageDefinition);

const server = new grpc.Server();

server.addService((usuarioProto.usuario as any).UsuarioService.service, {
  crear: (call: any, callback: any) => {
    const { nombre, email } = call.request;
    callback(null, {
      id: "123",
      nombre,
      email,
      creado_en: new Date().toISOString(),
    });
  },
});

server.bindAsync("0.0.0.0:50051", grpc.ServerCredentials.createInsecure(), () => {
  console.log("gRPC server en :50051");
});
```

---

## 15.7 Server-Sent Events

```typescript
// SSE: streaming de servidor a cliente
import { Request, Response } from "express";

function sseHandler(req: Request, res: Response) {
  res.writeHead(200, {
    "Content-Type": "text/event-stream",
    "Cache-Control": "no-cache",
    Connection: "keep-alive",
  });

  const interval = setInterval(() => {
    const data = JSON.stringify({
      hora: new Date().toISOString(),
      valor: Math.floor(Math.random() * 100),
    });
    res.write(`data: ${data}\n\n`);
  }, 1000);

  req.on("close", () => {
    clearInterval(interval);
  });
}
```

---

## 15.8 node:crypto y WebCrypto

```typescript
import { createHash, randomBytes, scrypt, timingSafeEqual, createCipheriv, createDecipheriv } from "node:crypto";
import { promisify } from "node:util";

const scryptAsync = promisify(scrypt);

// Hashing (irreversible)
function hashSHA256(datos: string): string {
  return createHash("sha256").update(datos).digest("hex");
}

console.log(hashSHA256("password123"));
// "ef92b778bafe771e89245b89ecbc..."

// Hashing de password con salt (scrypt)
async function hashearPassword(password: string): Promise<string> {
  const salt = randomBytes(16).toString("hex");
  const derivedKey = await scryptAsync(password, salt, 64) as Buffer;
  return `${salt}:${derivedKey.toString("hex")}`;
}

async function verificarPassword(password: string, hash: string): Promise<boolean> {
  const [salt, key] = hash.split(":");
  const derivedKey = await scryptAsync(password, salt, 64) as Buffer;
  return timingSafeEqual(Buffer.from(key, "hex"), derivedKey);
}

// Encriptacion simetrica (AES-256-GCM)
const ALGORITMO = "aes-256-gcm";
const clave = randomBytes(32);
const iv = randomBytes(16);

function encriptar(texto: string): { encriptado: string; iv: string; tag: string } {
  const cipher = createCipheriv(ALGORITMO, clave, iv);
  let encriptado = cipher.update(texto, "utf8", "hex");
  encriptado += cipher.final("hex");
  return { encriptado, iv: iv.toString("hex"), tag: cipher.getAuthTag().toString("hex") };
}

function desencriptar(encriptado: string, iv: string, tag: string): string {
  const decipher = createDecipheriv(ALGORITMO, clave, Buffer.from(iv, "hex"));
  decipher.setAuthTag(Buffer.from(tag, "hex"));
  let descifrado = decipher.update(encriptado, "hex", "utf8");
  descifrado += decipher.final("utf8");
  return descifrado;
}

// WebCrypto: API estandar para navegador Y Node.js
async function hashWebCrypto(datos: string): Promise<string> {
  const encoder = new TextEncoder();
  const hashBuffer = await crypto.subtle.digest("SHA-256", encoder.encode(datos));
  return Buffer.from(hashBuffer).toString("hex");
}
```

---

## 15.9 fetch en Node.js

```typescript
// fetch nativo en Node.js 18+ (basado en undici)
const respuesta = await fetch("https://api.github.com/repos/nodejs/node");

console.log(respuesta.status);            // 200
console.log(respuesta.headers.get("content-type"));

// Streaming del body (no cargar todo en memoria)
const reader = respuesta.body!.getReader();
const decoder = new TextDecoder();

while (true) {
  const { done, value } = await reader.read();
  if (done) break;
  console.log(decoder.decode(value, { stream: true }));
}

// fetch con AbortSignal (timeout)
const controller = new AbortController();
setTimeout(() => controller.abort(), 5000);

const datos = await fetch("https://api.ejemplo.com", {
  signal: controller.signal,
  method: "POST",
  headers: { "Content-Type": "application/json" },
  body: JSON.stringify({ nombre: "Andres" }),
}).then((r) => r.json());

// Subir archivo con FormData
const formData = new FormData();
formData.append("archivo", new Blob(["contenido"]), "archivo.txt");
formData.append("nombre", "Andres");

const upload = await fetch("/api/upload", { method: "POST", body: formData });
```

---

## 15.10 HTTP/2 con node:http2

```typescript
import { createSecureServer } from "node:http2";
import { readFileSync } from "node:fs";

const server = createSecureServer({
  key: readFileSync("localhost-key.pem"),
  cert: readFileSync("localhost-cert.pem"),
});

server.on("stream", (stream, headers) => {
  const method = headers[":method"];
  const path = headers[":path"];

  console.log(`${method} ${path}`);

  stream.respond({
    "content-type": "application/json",
    ":status": 200,
  });

  stream.end(JSON.stringify({ mensaje: "HTTP/2!", path, method }));
});

server.listen(8443);

// Cliente HTTP/2
import { connect } from "node:http2";

const client = connect("https://localhost:8443");
const req = client.request({ ":path": "/api/usuarios" });

req.setEncoding("utf8");
let data = "";
req.on("data", (chunk) => (data += chunk));
req.on("end", () => {
  console.log(JSON.parse(data));
  client.close();
});
req.end();
```

---

## 15.11 BullMQ: Colas de Trabajo

```bash
pnpm add bullmq
```

```typescript
import { Queue, Worker, QueueEvents } from "bullmq";

const connection = { host: "localhost", port: 6379 };

// Productor: encolar trabajos
const colaEmails = new Queue("emails", { connection });

await colaEmails.add("bienvenida", {
  email: "usuario@test.com",
  nombre: "Andres",
  template: "welcome",
});

await colaEmails.add("recuperacion", {
  email: "otro@test.com",
  token: "reset-abc",
}, {
  delay: 5000,           // Ejecutar en 5s
  attempts: 3,           // Reintentar hasta 3 veces
  backoff: { type: "exponential", delay: 1000 },
});

// Consumidor: procesar trabajos
const worker = new Worker("emails", async (job) => {
  const { email, nombre, template } = job.data;
  console.log(`Enviando ${template} a ${nombre} (${email})`);

  await job.updateProgress(50);
  // ... enviar email real ...
  await job.updateProgress(100);

  return { enviado: true, timestamp: Date.now() };
}, { connection, concurrency: 5 });

// Monitorear eventos
const eventos = new QueueEvents("emails", { connection });
eventos.on("completed", ({ jobId, returnvalue }) => {
  console.log(`Job ${jobId} completado:`, returnvalue);
});
eventos.on("failed", ({ jobId, failedReason }) => {
  console.error(`Job ${jobId} fallo:`, failedReason);
});
```

---

## 15.12 node:test (Test Runner Nativo)

```typescript
import { describe, it, before, after } from "node:test";
import assert from "node:assert/strict";

describe("API de usuarios", () => {
  let server;

  before(async () => {
    server = await iniciarServidorTest();
  });

  after(async () => {
    await server.close();
  });

  it("GET /api/health retorna ok", async () => {
    const res = await fetch("http://localhost:3000/api/health");
    assert.equal(res.status, 200);
    const body = await res.json();
    assert.deepEqual(body, { status: "ok" });
  });

  it("POST /api/usuarios crea usuario", async () => {
    const res = await fetch("http://localhost:3000/api/usuarios", {
      method: "POST",
      body: JSON.stringify({ nombre: "Test", email: "test@test.com" }),
    });
    assert.equal(res.status, 201);
  });
});
```

```bash
# Ejecutar con node:test
node --test src/**/*.test.ts
node --test --test-reporter spec src/**/*.test.ts
node --test --experimental-test-coverage src/**/*.test.ts
```

### Dead Letter Queue (DLQ) y Poison Messages

```typescript
import { Queue, Worker, QueueEvents } from "bullmq";

// Cola principal con DLQ configurada
const colaEmails = new Queue("emails", {
  connection,
  defaultJobOptions: {
    attempts: 3,                     // Reintentar maximo 3 veces
    backoff: { type: "exponential", delay: 5000 },
    removeOnComplete: { age: 3600 }, // Limpiar completados despues de 1h
    removeOnFail: false,             // Mantener fallidos para inspeccionar
  },
});

// Worker: si falla 3 veces, va al DLQ
const worker = new Worker("emails", async (job) => {
  const { email, template } = job.data;
  await enviarEmail(email, template);
}, { connection });

// DLQ: cola separada para mensajes fallidos
const dlqEmails = new Queue("emails-dlq", { connection });

// Escuchar fallos y mover a DLQ manualmente si es necesario
const eventos = new QueueEvents("emails", { connection });
eventos.on("failed", async ({ jobId, failedReason }) => {
  const job = await colaEmails.getJob(jobId);
  if (job && job.attemptsMade >= job.opts.attempts!) {
    // Movido a DLQ despues de agotar reintentos
    await dlqEmails.add("email-fallido", job.data, {
      removeOnComplete: false,
    });
  }
});

// Monitorear DLQ: alertas si crece
setInterval(async () => {
  const count = await dlqEmails.getJobCounts();
  if (count.waiting && count.waiting > 10) {
    logger.error("DLQ creciendo!", { count: count.waiting });
    // Enviar alerta a on-call, revisar manualmente
  }
}, 60_000);
```

---

## 15.13 Detectar Event Loop Bloqueado

```typescript
// blocked-at: detecta cuando el event loop se bloquea > N ms
import blocked from "blocked-at";

blocked((time, stack) => {
  logger.error(`Event loop bloqueado por ${time}ms`, { stack });
}, { threshold: 50 }); // Alerta si se bloquea > 50ms

// under-pressure: monitorea salud del event loop
import { monitorEventLoopDelay } from "node:perf_hooks";

const histogram = monitorEventLoopDelay();
histogram.enable();

setInterval(() => {
  console.log({
    min: histogram.min / 1e6,
    max: histogram.max / 1e6,
    mean: histogram.mean / 1e6,
    p99: histogram.percentile(99) / 1e6,
  });
}, 5000);
```

### Lo que bloquea el Event Loop

```typescript
// ❌ BLOQUEO: JSON.parse en payloads grandes (>1MB)
app.post("/api/bulk", (req, res) => {
  let body = "";
  req.on("data", (chunk) => (body += chunk));
  req.on("end", () => {
    const data = JSON.parse(body); // Bloquea el loop!
    // Solucion: stream JSON parser (jsonstream, stream-json)
  });
});

// ❌ BLOQUEO: Regex con backtracking explosivo (ReDoS)
const emailRegex = /^([a-zA-Z0-9]\.?)+@([a-zA-Z0-9]\.?)+\.([a-zA-Z]{2,})+$/;
// Entrada maliciosa: "aaaaaaaaaaaaaaaaaaaaaaaa!" -> minutos de CPU
// Solucion: regex segura, timeout, validator.js

// ❌ BLOQUEO: crypto sincrono en grandes datos
const hash = crypto.createHash("sha256").update(hugeBuffer).digest("hex");
// Solucion: crypto asincrono o Worker Thread

// ❌ BLOQUEO: iterar arrays enormes sincronamente
for (const item of arrayDeMillones) {
  item.procesarSync(); // Bloquea el loop mientras itera
}
// Solucion: partir en chunks con setImmediate o Worker Thread
```

---

## 15.14 Memory Leaks: Deteccion y Patrones

```typescript
// Patron de fuga #1: Event listeners sin limpiar
class MiServicio {
  private emitter = new EventEmitter();

  start() {
    // ❌ Fuga: cada llamada a start() agrega un listener
    this.emitter.on("datos", this.procesarDatos);
  }

  stop() {
    // ✅ Arreglo: remover listener al detener
    this.emitter.off("datos", this.procesarDatos);
  }
}

// Patron de fuga #2: setInterval sin limpiar
class Monitor {
  private interval?: NodeJS.Timeout;

  start() {
    this.interval = setInterval(() => {
      this.chequear(); // Si monitor se destruye, esto sigue corriendo
    }, 1000);
  }

  stop() {
    if (this.interval) clearInterval(this.interval);
  }
}

// Patron de fuga #3: Cache sin limite de tamaño
// ❌ cache global que crece sin bound
const cacheGlobal = new Map<string, any>();

// ✅ LRU cache con max size
import { LRUCache } from "lru-cache";
const cache = new LRUCache<string, any>({
  max: 1000,               // Maximo 1000 entradas
  ttl: 5 * 60 * 1000,     // 5 minutos TTL
});

// Patron de fuga #4: Closures que retienen referencias grandes
function crearHandler() {
  const datosGrandes = cargarDatosEnormes(); // 500MB

  return (req, res) => {
    // ❌ Este closure retiene datosGrandes para siempre
    res.json({ mensaje: "ok" });
  };
}

// ✅ Solo capturar lo necesario
function crearHandler() {
  return (req, res) => {
    res.json({ mensaje: "ok" }); // Nada grande capturado
  };
}
```

### Debugging de Memory Leaks

```bash
# Tomar heap snapshot
node --heapsnapshot-near-heap-limit=3 dist/index.js

# O manualmente con Chrome DevTools
node --inspect dist/index.js
# Chrome -> Memory -> Take snapshot -> operar -> Take snapshot -> Comparison view

# Detectar fugas en tests
node --expose-gc -e "
  const obj = [];
  for (let i = 0; i < 1000000; i++) obj.push({});
  global.gc();
  console.log(process.memoryUsage().heapUsed / 1024 / 1024, 'MB');
"
```

---

## 15.15 V8 Internals: Lo que Todo Dev Node.js Debe Saber

### Hidden Classes y Inline Caching

```typescript
// V8 optimiza objetos con la misma "forma" (mismas propiedades en el mismo orden)

// ✅ MISMA hidden class -> optimizado
const a = { x: 1, y: 2 };
const b = { x: 3, y: 4 };
// a y b comparten hidden class -> acceso rapido

// ❌ DIFERENTE hidden class -> deoptimiza
const c = { x: 1, y: 2 };
const d = { y: 2, x: 1 }; // Orden diferente!
// c y d tienen diferentes hidden classes

// ❌ Agregar/eliminar propiedades cambia la hidden class
const obj = { x: 1 };
obj.y = 2;  // Cambia la hidden class
delete obj.x; // MEGA deoptimizacion: evita delete en objetos calientes

// Regla: inicializa todas las propiedades en el constructor
class Usuario {
  nombre = "";    // Inicializar todas
  email = "";     // aunque sea con defaults
  edad = 0;
  // No agregar propiedades fuera del constructor
}
```

### Monomorfismo vs Polimorfismo vs Megamorfismo

```typescript
// V8 hace inline caching: recuerda el tipo de objeto que vio la funcion

function obtenerNombre(obj: { nombre: string }) {
  return obj.nombre; // V8 cachea el "acceso rapido" a .nombre
}

// MONOMORFICO (1 tipo): MAS RAPIDO
obtenerNombre({ nombre: "Andres" });
obtenerNombre({ nombre: "Maria" });

// POLIMORFICO (2-4 tipos): ACEPTABLE
obtenerNombre({ nombre: "Andres" });
// obtenerNombre({ name: "John" }); // Tipo 2
// obtenerNombre({ nombre_completo: "Carlos" }); // Tipo 3

// MEGAMORFICO (>4 tipos): DEOPTIMIZA COMPLETO
// V8 se rinde y usa diccionario (mas lento)

// Regla: usa interfaces consistentes, no mezcles formas de objeto
```

### try/catch y deoptimizacion

```typescript
// ❌ try/catch en funciones calientes DEOPTIMIZA
function procesarLote(items: number[]) {
  for (const item of items) {
    try {
      procesarItem(item); // Funcion caliente con try/catch = lenta
    } catch { /* ... */ }
  }
}

// ✅ Mover try/catch fuera del bucle caliente
function procesarLote(items: number[]) {
  for (const item of items) {
    procesarItem(item); // Sin try/catch = rapido
  }
}

function procesarItem(item: number) {
  if (item < 0) return; // Validacion sin try/catch
  // Procesar...
}
```

---

## 15.16 libuv Thread Pool y Starvation

```typescript
// libuv tiene 4 hilos por defecto para operaciones bloqueantes:
// fs (algunas), crypto (algunas), dns.lookup, zlib, gzip

// Sintoma de saturacion: operaciones rapidas se vuelven lentas
// porque esperan en la cola del thread pool

// Detectar saturacion
import { hrtime } from "node:process";

const start = hrtime.bigint();
await fs.readFile("/tmp/test.txt");
const end = hrtime.bigint();
console.log(`readFile: ${Number(end - start) / 1e6}ms`);
// Si >100ms para archivos pequeños, el pool esta saturado

// Ajustar thread pool
// UV_THREADPOOL_SIZE=16 node dist/index.js

// Que operaciones NO usan el thread pool (son async nativas del SO):
// - Red (TCP/UDP): epoll/kqueue/IOCP
// - Resolucion DNS via c-ares (dns.resolve)
// - Señales, timers, eventos de proceso
```

---

## Resumen del Capítulo

- **Decorators**: TC39 Stage 3 para metaprogramacion (logging, DI, validacion).
- **Streams**: procesamiento eficiente de grandes volumenes. Custom Readable/Transform/Writable.
- **Performance**: `performance.now()` para mediciones, Clinic.js para profiling de produccion.
- **Cluster/PM2**: aprovecha todos los cores con cluster mode. PM2 para gestion de procesos.
- **Seguridad**: Helmet, rate limiting, Zod validation, nunca exponer errores internos.
- **gRPC**: comunicacion rapida entre servicios con Protobuf y tipado fuerte.
- **SSE**: streaming unidireccional servidor->cliente sin WebSockets.

En el siguiente capítulo exploraremos el desarrollo web y APIs REST en profundidad.

---

← [Capítulo anterior](14-arquitectura-hexagonal.md) | [Inicio](README.md) | [Capítulo siguiente →](16-desarrollo-web.md)
