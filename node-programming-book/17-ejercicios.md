# Capítulo 17: Ejercicios Prácticos

Este capitulo contiene ejercicios organizados por nivel de dificultad para practicar y consolidar los conceptos de Node.js y TypeScript. Incluye descripcion, pistas y solucion comentada.

---

## 17.1 Ejercicios Basicos

### Ejercicio 1: CLI de Calculadora

Crea un programa CLI que reciba dos numeros y un operador y muestre el resultado.

```typescript
// solucion/calculadora.ts
function calcular(a: number, operador: string, b: number): number {
  switch (operador) {
    case "+": return a + b;
    case "-": return a - b;
    case "*": return a * b;
    case "/":
      if (b === 0) throw new Error("Division por cero");
      return a / b;
    default:
      throw new Error(`Operador desconocido: ${operador}`);
  }
}

const [, , aStr, operador, bStr] = process.argv;
const a = Number(aStr);
const b = Number(bStr);

if (isNaN(a) || isNaN(b)) {
  console.error("Error: Ambos argumentos deben ser numeros");
  process.exit(1);
}

try {
  console.log(calcular(a, operador, b));
} catch (error) {
  console.error((error as Error).message);
  process.exit(1);
}
```

### Ejercicio 2: Servidor HTTP Simple

Crea un servidor HTTP con 3 rutas: `GET /`, `GET /saludo?nombre=X`, `POST /eco`.

```typescript
import { createServer, IncomingMessage, ServerResponse } from "node:http";

const server = createServer(async (req, res) => {
  const url = new URL(req.url!, `http://${req.headers.host}`);

  if (url.pathname === "/" && req.method === "GET") {
    res.writeHead(200, { "Content-Type": "text/plain" });
    return res.end("Bienvenido!");
  }

  if (url.pathname === "/saludo" && req.method === "GET") {
    const nombre = url.searchParams.get("nombre") ?? "Mundo";
    res.writeHead(200, { "Content-Type": "application/json" });
    return res.end(JSON.stringify({ mensaje: `Hola, ${nombre}!` }));
  }

  if (url.pathname === "/eco" && req.method === "POST") {
    let body = "";
    req.on("data", (chunk) => (body += chunk));
    req.on("end", () => {
      res.writeHead(200, { "Content-Type": "application/json" });
      res.end(JSON.stringify({ recibido: JSON.parse(body) }));
    });
    return;
  }

  res.writeHead(404);
  res.end("No encontrado");
});

server.listen(3000, () => console.log("Servidor en :3000"));
```

---

## 17.2 Ejercicios Intermedios

### Ejercicio 3: API REST de Tareas

Construye una API REST para gestionar tareas con Express y Zod.

```typescript
// solucion/task-api.ts
import express, { Request, Response } from "express";
import { z } from "zod";
import { randomUUID } from "node:crypto";

interface Tarea {
  id: string;
  titulo: string;
  completada: boolean;
}

const tareas: Tarea[] = [];

const crearSchema = z.object({ titulo: z.string().min(1) });
const actualizarSchema = z.object({
  titulo: z.string().min(1).optional(),
  completada: z.boolean().optional(),
});

const app = express();
app.use(express.json());

app.get("/api/tareas", (_req, res) => res.json(tareas));

app.post("/api/tareas", (req, res) => {
  const { titulo } = crearSchema.parse(req.body);
  const tarea: Tarea = { id: randomUUID(), titulo, completada: false };
  tareas.push(tarea);
  res.status(201).json(tarea);
});

app.put("/api/tareas/:id", (req, res) => {
  const data = actualizarSchema.parse(req.body);
  const tarea = tareas.find((t) => t.id === req.params.id);
  if (!tarea) return res.status(404).json({ error: "No encontrada" });

  if (data.titulo !== undefined) tarea.titulo = data.titulo;
  if (data.completada !== undefined) tarea.completada = data.completada;
  res.json(tarea);
});

app.delete("/api/tareas/:id", (req, res) => {
  const idx = tareas.findIndex((t) => t.id === req.params.id);
  if (idx === -1) return res.status(404).json({ error: "No encontrada" });
  tareas.splice(idx, 1);
  res.sendStatus(204);
});

app.listen(3000, () => console.log("API en :3000"));
```

### Ejercicio 4: Cache con TTL

Implementa una cache generica con tiempo de expiracion.

```typescript
class Cache<T> {
  private store = new Map<string, { value: T; expiresAt: number }>();
  private interval: NodeJS.Timeout;

  constructor(private ttlMs: number = 60_000) {
    this.interval = setInterval(() => this.limpiarExpirados(), ttlMs);
  }

  set(key: string, value: T): void {
    this.store.set(key, { value, expiresAt: Date.now() + this.ttlMs });
  }

  get(key: string): T | undefined {
    const entry = this.store.get(key);
    if (!entry) return undefined;
    if (Date.now() > entry.expiresAt) {
      this.store.delete(key);
      return undefined;
    }
    return entry.value;
  }

  delete(key: string): void { this.store.delete(key); }

  size(): number { return this.store.size; }

  private limpiarExpirados(): void {
    const ahora = Date.now();
    for (const [key, entry] of this.store) {
      if (ahora > entry.expiresAt) this.store.delete(key);
    }
  }

  destroy(): void { clearInterval(this.interval); }
}

// Uso
const cache = new Cache<string>(5000);
cache.set("user:1", JSON.stringify({ nombre: "Andres" }));
console.log(cache.get("user:1")); // { nombre: "Andres" }
setTimeout(() => console.log(cache.get("user:1")), 6000); // undefined
```

---

## 17.3 Ejercicios Avanzados

### Ejercicio 5: Rate Limiter con Token Bucket

```typescript
class RateLimiter {
  private buckets = new Map<string, { tokens: number; lastRefill: number }>();

  constructor(
    private maxTokens: number,
    private refillRate: number, // tokens por segundo
    private refillInterval: number, // ms entre refill
  ) {
    setInterval(() => this.refillAll(), this.refillInterval);
  }

  allow(key: string): boolean {
    let bucket = this.buckets.get(key);
    if (!bucket) {
      bucket = { tokens: this.maxTokens, lastRefill: Date.now() };
      this.buckets.set(key, bucket);
    }

    if (bucket.tokens > 0) {
      bucket.tokens--;
      return true;
    }
    return false;
  }

  private refillAll(): void {
    for (const bucket of this.buckets.values()) {
      bucket.tokens = Math.min(this.maxTokens, bucket.tokens + this.refillRate);
    }
  }
}

// Express middleware
function rateLimiter(limiter: RateLimiter) {
  return (req: express.Request, res: express.Response, next: express.NextFunction) => {
    const key = req.ip ?? "unknown";
    if (!limiter.allow(key)) {
      return res.status(429).json({ error: "Demasiadas peticiones" });
    }
    next();
  };
}
```

### Ejercicio 6: Chat en Tiempo Real con WebSocket

```typescript
import { WebSocketServer, WebSocket } from "ws";

const wss = new WebSocketServer({ port: 8080 });

const clientes = new Map<WebSocket, string>();

wss.on("connection", (ws) => {
  const id = `user-${Math.random().toString(36).slice(2, 8)}`;
  clientes.set(ws, id);

  ws.send(JSON.stringify({ tipo: "sistema", mensaje: `Conectado como ${id}` }));

  ws.on("message", (data) => {
    const mensaje = data.toString();

    for (const [cliente, clienteId] of clientes) {
      if (cliente.readyState === WebSocket.OPEN) {
        cliente.send(JSON.stringify({
          tipo: "mensaje",
          de: id,
          mensaje,
          timestamp: new Date().toISOString(),
        }));
      }
    }
  });

  ws.on("close", () => {
    clientes.delete(ws);
    for (const [cliente] of clientes) {
      cliente.send(JSON.stringify({ tipo: "sistema", mensaje: `${id} se desconecto` }));
    }
  });
});

console.log("Chat server en ws://localhost:8080");
```

---

## 17.4 Proyectos Integradores

### Proyecto 1: Microservicio de Tareas

Construye un microservicio completo con:
- CRUD de tareas con Express + TypeScript
- Zod validation en todos los endpoints
- Prisma ORM con PostgreSQL
- JWT authentication con login/register
- Tests unitarios con Vitest + tests de integracion con supertest
- Rate limiting, Helmet, CORS
- Dockerfile y docker-compose con PostgreSQL

### Proyecto 2: Acortador de URLs

Construye un servicio con:
- API REST para crear y redirigir URLs cortas
- Cache en Redis para URLs frecuentes
- Estadisticas de visitas (clicks por dia)
- URLs personalizadas y expiracion
- Rate limiting por IP

### Proyecto 3: Sistema de Eventos con Event Sourcing

Construye un sistema que:
- Publique eventos a un Event Bus
- Proyecte (proyecte = derive estado) a una base de datos de lectura
- Permita reconstruir el estado desde los eventos
- Use arquitectura hexagonal con dominio puro

---

## Resumen del Capítulo

- Los ejercicios basicos refuerzan HTTP nativo, CLI y parseo de entrada.
- Los ejercicios intermedios introducen APIs REST, Zod validation y patrones de cache.
- Los ejercicios avanzados combinan rate limiting, WebSockets y concurrencia.
- Los proyectos integradores consolidan todos los conceptos en aplicaciones completas.
- La practica constante es la clave. Intenta cada ejercicio antes de mirar la solucion.

---

Este es el final del libro. Has recorrido un camino completo desde los fundamentos de Node.js y TypeScript hasta arquitectura hexagonal, testing avanzado y desarrollo web profesional. El ecosistema Node.js + TypeScript es inmenso y sigue creciendo. Tu siguiente paso es construir proyectos reales, contribuir al open source y nunca dejar de aprender. ¡Feliz programacion!

---

← [Capítulo anterior](16-desarrollo-web.md) | [Inicio](README.md)
