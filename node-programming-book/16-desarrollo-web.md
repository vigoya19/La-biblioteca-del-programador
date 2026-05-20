# Capítulo 16: Desarrollo Web y APIs

Node.js es la plataforma dominante para APIs web. Este capitulo cubre desde HTTP nativo hasta frameworks como Express, Fastify y NestJS, incluyendo bases de datos, autenticacion y patrones de API REST.

---

## 16.1 HTTP Nativo en Node.js

```typescript
import { createServer, IncomingMessage, ServerResponse } from "node:http";

interface Ruta {
  metodo: string;
  path: RegExp;
  handler: (req: IncomingMessage, res: ServerResponse, params: Record<string, string>) => void;
}

const rutas: Ruta[] = [];

function get(path: string, handler: Ruta["handler"]) {
  rutas.push({ metodo: "GET", path: new RegExp(`^${path}$`), handler });
}

function post(path: string, handler: Ruta["handler"]) {
  rutas.push({ metodo: "POST", path: new RegExp(`^${path}$`), handler });
}

async function parseBody(req: IncomingMessage): Promise<unknown> {
  return new Promise((resolve) => {
    let body = "";
    req.on("data", (chunk) => (body += chunk));
    req.on("end", () => resolve(body ? JSON.parse(body) : {}));
  });
}

const server = createServer(async (req, res) => {
  const url = new URL(req.url!, `http://${req.headers.host}`);

  for (const ruta of rutas) {
    const match = url.pathname.match(ruta.path);
    if (match && ruta.metodo === req.method) {
      const params = match.groups ?? {};
      await ruta.handler(req, res, params);
      return;
    }
  }

  res.writeHead(404, { "Content-Type": "application/json" });
  res.end(JSON.stringify({ error: "No encontrado" }));
});

get("/api/usuarios", async (req, res) => {
  res.writeHead(200, { "Content-Type": "application/json" });
  res.end(JSON.stringify([{ id: 1, nombre: "Andres" }]));
});

server.listen(8080, () => console.log("Servidor en :8080"));
```

---

## 16.2 Express.js en Profundidad

```typescript
import express, { Request, Response, NextFunction } from "express";
import helmet from "helmet";
import cors from "cors";
import { z } from "zod";

const app = express();

// ─── Middlewares globales ───
app.use(helmet());
app.use(cors());
app.use(express.json());

// ─── Middleware personalizado ───
function loggingMiddleware(req: Request, res: Response, next: NextFunction) {
  const inicio = Date.now();
  res.on("finish", () => {
    console.log(`${req.method} ${req.path} ${res.statusCode} ${Date.now() - inicio}ms`);
  });
  next();
}

app.use(loggingMiddleware);

// ─── Validacion con Zod ───
const crearUsuarioSchema = z.object({
  body: z.object({
    email: z.string().email(),
    nombre: z.string().min(1).max(100),
  }),
});

function validate(schema: z.ZodSchema) {
  return (req: Request, res: Response, next: NextFunction) => {
    const result = schema.safeParse({ body: req.body, query: req.query, params: req.params });
    if (!result.success) {
      return res.status(400).json({ error: result.error.flatten() });
    }
    next();
  };
}

// ─── Rutas ───
app.get("/api/health", (req, res) => {
  res.json({ status: "ok", timestamp: new Date().toISOString() });
});

app.post("/api/usuarios", validate(crearUsuarioSchema), async (req, res) => {
  const { email, nombre } = req.body;
  // ... crear usuario ...
  res.status(201).json({ id: 1, email, nombre });
});

// ─── Error handler ───
app.use((err: Error, req: Request, res: Response, next: NextFunction) => {
  console.error("Error:", err.message);
  res.status(500).json({ error: "Error interno del servidor" });
});

app.listen(8080, () => console.log("Express en :8080"));
```

---

## 16.3 Fastify: Rendimiento y Schema Validation

```typescript
import Fastify from "fastify";

const app = Fastify({ logger: true });

// Schema validation nativa (JSON Schema)
app.post("/api/usuarios", {
  schema: {
    body: {
      type: "object",
      required: ["email", "nombre"],
      properties: {
        email: { type: "string", format: "email" },
        nombre: { type: "string", minLength: 1, maxLength: 100 },
      },
    },
    response: {
      201: {
        type: "object",
        properties: {
          id: { type: "number" },
          email: { type: "string" },
          nombre: { type: "string" },
        },
      },
    },
  },
}, async (request, reply) => {
  const { email, nombre } = request.body as any;
  // Fastify serializa JSON 2x mas rapido que Express
  reply.code(201).send({ id: 1, email, nombre });
});

// Hooks (middleware-style)
app.addHook("onRequest", async (request) => {
  request.log.info(`Peticion entrante: ${request.method} ${request.url}`);
});

// Decorators (extender Fastify)
app.decorate("db", prisma);

await app.listen({ port: 8080 });
```

---

## 16.4 Bases de Datos

### Prisma ORM

```prisma
// prisma/schema.prisma
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

model Usuario {
  id        String   @id @default(uuid())
  email     String   @unique
  nombre    String
  activo    Boolean  @default(true)
  creadoEn  DateTime @default(now()) @map("creado_en")
  pedidos   Pedido[]

  @@map("usuarios")
}

model Pedido {
  id        String   @id @default(uuid())
  total     Float
  usuarioId String   @map("usuario_id")
  usuario   Usuario  @relation(fields: [usuarioId], references: [id])
  creadoEn  DateTime @default(now()) @map("creado_en")

  @@map("pedidos")
}
```

```typescript
import { PrismaClient } from "@prisma/client";

const prisma = new PrismaClient();

// Crear
const usuario = await prisma.usuario.create({
  data: { email: "test@test.com", nombre: "Andres" },
});

// Leer con relaciones
const usuarioConPedidos = await prisma.usuario.findUnique({
  where: { id: usuario.id },
  include: { pedidos: true },
});

// Transaccion
await prisma.$transaction(async (tx) => {
  const u = await tx.usuario.create({ data: { email, nombre } });
  await tx.pedido.create({ data: { total: 99.99, usuarioId: u.id } });
});
```

### Drizzle ORM (SQL-like, ligero)

```typescript
import { pgTable, text, boolean, timestamp, uuid } from "drizzle-orm/pg-core";
import { drizzle } from "drizzle-orm/node-postgres";

const usuarios = pgTable("usuarios", {
  id: uuid("id").defaultRandom().primaryKey(),
  email: text("email").unique().notNull(),
  nombre: text("nombre").notNull(),
  activo: boolean("activo").default(true).notNull(),
  creadoEn: timestamp("creado_en").defaultNow().notNull(),
});

const db = drizzle(process.env.DATABASE_URL!);

// SQL-like
const adultos = await db
  .select()
  .from(usuarios)
  .where(eq(usuarios.activo, true))
  .limit(10);
```

---

## 16.5 Autenticacion

### JWT (JSON Web Tokens)

```typescript
import jwt from "jsonwebtoken";

const JWT_SECRET = process.env.JWT_SECRET!;

// Generar token
function generarToken(usuarioId: string): string {
  return jwt.sign(
    { sub: usuarioId, iat: Math.floor(Date.now() / 1000) },
    JWT_SECRET,
    { expiresIn: "24h" },
  );
}

// Verificar token
function verificarToken(token: string): { sub: string } {
  return jwt.verify(token, JWT_SECRET) as { sub: string };
}

// Middleware de auth
import { Request, Response, NextFunction } from "express";

interface AuthRequest extends Request {
  usuarioId?: string;
}

function authMiddleware(req: AuthRequest, res: Response, next: NextFunction) {
  const header = req.headers.authorization;
  if (!header?.startsWith("Bearer ")) {
    return res.status(401).json({ error: "Token requerido" });
  }
  try {
    const payload = verificarToken(header.slice(7));
    req.usuarioId = payload.sub;
    next();
  } catch {
    res.status(401).json({ error: "Token invalido" });
  }
}
```

### bcrypt para passwords

```typescript
import bcrypt from "bcrypt";

const SALT_ROUNDS = 12;

async function hashearPassword(password: string): Promise<string> {
  return bcrypt.hash(password, SALT_ROUNDS);
}

async function verificarPassword(
  password: string,
  hash: string,
): Promise<boolean> {
  return bcrypt.compare(password, hash);
}
```

---

## 16.6 File Uploads

```typescript
import multer from "multer";
import path from "node:path";

const storage = multer.diskStorage({
  destination: "uploads/",
  filename: (req, file, cb) => {
    const unique = `${Date.now()}-${Math.round(Math.random() * 1e9)}`;
    cb(null, `${unique}${path.extname(file.originalname)}`);
  },
});

const upload = multer({
  storage,
  limits: { fileSize: 5 * 1024 * 1024 }, // 5MB
  fileFilter: (req, file, cb) => {
    const permitidos = /jpeg|jpg|png|webp/;
    const ok = permitidos.test(path.extname(file.originalname));
    cb(null, ok);
  },
});

app.post("/api/upload", upload.single("archivo"), (req, res) => {
  if (!req.file) return res.status(400).json({ error: "Archivo requerido" });
  res.json({ url: `/uploads/${req.file.filename}` });
});
```

---

## 16.7 API REST: Diseno Completo

```typescript
// Entidad
interface Tarea {
  id: string;
  titulo: string;
  descripcion: string;
  completada: boolean;
  creadaEn: Date;
}

// Repositorio (puerto)
interface TareaRepository {
  listar(): Promise<Tarea[]>;
  crear(data: Pick<Tarea, "titulo" | "descripcion">): Promise<Tarea>;
  actualizar(id: string, data: Partial<Tarea>): Promise<Tarea>;
  eliminar(id: string): Promise<void>;
}

// Servicio (caso de uso)
class TareaService {
  constructor(private repo: TareaRepository) {}

  async listar(): Promise<Tarea[]> { return this.repo.listar(); }
  async crear(data: { titulo: string; descripcion: string }) {
    if (!data.titulo.trim()) throw new Error("Titulo requerido");
    return this.repo.crear(data);
  }
}

// Routes
import { Router } from "express";

function crearTareaRouter(service: TareaService): Router {
  const router = Router();

  router.get("/tareas", async (req, res) => {
    const tareas = await service.listar();
    res.json({ data: tareas });
  });

  router.post("/tareas", async (req, res) => {
    try {
      const tarea = await service.crear(req.body);
      res.status(201).json({ data: tarea });
    } catch (error) {
      res.status(400).json({ error: (error as Error).message });
    }
  });

  return router;
}
```

### Paginacion y filtros

```typescript
interface PaginacionQuery {
  pagina?: number;
  limite?: number;
  orden?: string;
  direccion?: "asc" | "desc";
}

interface RespuestaPaginada<T> {
  data: T[];
  meta: {
    pagina: number;
    limite: number;
    total: number;
    totalPaginas: number;
  };
}

async function listarPaginado<T>(
  query: PaginacionQuery,
  contarFn: () => Promise<number>,
  listarFn: (offset: number, limit: number) => Promise<T[]>,
): Promise<RespuestaPaginada<T>> {
  const pagina = Math.max(1, query.pagina ?? 1);
  const limite = Math.min(100, Math.max(1, query.limite ?? 10));
  const offset = (pagina - 1) * limite;

  const [data, total] = await Promise.all([
    listarFn(offset, limite),
    contarFn(),
  ]);

  return {
    data,
    meta: {
      pagina,
      limite,
      total,
      totalPaginas: Math.ceil(total / limite),
    },
  };
}
```

---

## 16.8 NestJS: Arquitectura Opinionada

```bash
# Crear proyecto NestJS
npx @nestjs/cli new mi-api --package-manager pnpm
```

```typescript
// usuario.controller.ts
import { Controller, Get, Post, Body, Param, UseGuards, Req } from "@nestjs/common";
import { UsuarioService } from "./usuario.service";
import { AuthGuard } from "../auth/auth.guard";

@Controller("usuarios")
export class UsuarioController {
  constructor(private readonly service: UsuarioService) {}

  @Get()
  @UseGuards(AuthGuard)
  listar(@Req() req: { usuario: { id: string } }) {
    return this.service.listar();
  }

  @Get(":id")
  obtener(@Param("id") id: string) {
    return this.service.obtenerPorId(id);
  }

  @Post()
  async crear(@Body() dto: CrearUsuarioDTO) {
    return this.service.crear(dto);
  }
}

// Pipes: validacion y transformacion automatica
import { IsEmail, IsString, MinLength } from "class-validator";

export class CrearUsuarioDTO {
  @IsEmail()
  email: string;

  @IsString()
  @MinLength(1)
  nombre: string;
}

// Guards: proteccion de rutas
import { CanActivate, ExecutionContext } from "@nestjs/common";

@Injectable()
export class AuthGuard implements CanActivate {
  canActivate(context: ExecutionContext): boolean {
    const request = context.switchToHttp().getRequest();
    const token = request.headers.authorization?.split(" ")[1];
    request.usuario = verificarToken(token);
    return !!request.usuario;
  }
}

// Interceptors: transformar respuesta/envuelta
@Injectable()
export class ResponseInterceptor implements NestInterceptor {
  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    return next.handle().pipe(
      map((datos) => ({ data: datos, timestamp: new Date().toISOString() })),
    );
  }
}
```

---

## 16.9 tRPC: APIs Type-Safe End-to-End

```bash
pnpm add @trpc/server zod
```

```typescript
// server.ts: definicion de API
import { initTRPC } from "@trpc/server";
import { z } from "zod";

const t = initTRPC.create();

export const router = t.router;
export const publicProcedure = t.procedure;

const appRouter = router({
  saludar: publicProcedure
    .input(z.object({ nombre: z.string() }))
    .query(({ input }) => `Hola, ${input.nombre}!`),

  usuarios: router({
    listar: publicProcedure
      .input(z.object({ limite: z.number().min(1).max(100).default(10) }))
      .query(async ({ input }) => {
        return db.usuario.findMany({ take: input.limite });
      }),

    crear: publicProcedure
      .input(z.object({ email: z.string().email(), nombre: z.string().min(1) }))
      .mutation(async ({ input }) => {
        return db.usuario.create({ data: input });
      }),
  }),
});

export type AppRouter = typeof appRouter;
```

```typescript
// cliente.ts: consumo type-safe
import { createTRPCClient } from "@trpc/client";
import type { AppRouter } from "./server";

const cliente = createTRPCClient<AppRouter>({ url: "http://localhost:3000/trpc" });

// TypeScript infiere todo: input, output, errores
const saludo = await cliente.saludar.query({ nombre: "Andres" });
// saludo: string

const usuarios = await cliente.usuarios.listar.query({ limite: 5 });
// usuarios: Usuario[] (tipo inferido del schema de BD)

const nuevo = await cliente.usuarios.crear.mutate({
  email: "test@test.com",
  nombre: "Test",
});
// nuevo: Usuario
```

---

## 16.10 OAuth2 y Refresh Tokens

```typescript
// Server: OAuth2 con PKCE
interface TokenPayload {
  sub: string;   // usuario ID
  type: "access" | "refresh";
  iat: number;
  exp: number;
}

function generarAccessToken(usuarioId: string): string {
  return jwt.sign({ sub: usuarioId, type: "access" }, JWT_ACCESS_SECRET, { expiresIn: "15m" });
}

function generarRefreshToken(usuarioId: string): string {
  return jwt.sign({ sub: usuarioId, type: "refresh" }, JWT_REFRESH_SECRET, { expiresIn: "7d" });
}

// Refresh token rotation: invalidar token anterior al usarlo
async function refrescarToken(refreshToken: string): Promise<{ access: string; refresh: string }> {
  const payload = jwt.verify(refreshToken, JWT_REFRESH_SECRET) as TokenPayload;
  const tokenEnDB = await db.refreshToken.findUnique({ where: { token: refreshToken } });
  if (!tokenEnDB || tokenEnDB.usado) throw new Error("Token invalido");

  // Marcar como usado (rotation)
  await db.refreshToken.update({ where: { id: tokenEnDB.id }, data: { usado: true } });

  // Generar nuevo par
  const nuevoAccess = generarAccessToken(payload.sub);
  const nuevoRefresh = generarRefreshToken(payload.sub);
  await db.refreshToken.create({ data: { token: nuevoRefresh, usuarioId: payload.sub } });

  return { access: nuevoAccess, refresh: nuevoRefresh };
}
```

---

## 16.11 GraphQL con Apollo + TypeGraphQL

```typescript
import { ObjectType, Field, ID, Query, Resolver, Mutation, Arg, buildSchema } from "type-graphql";

@ObjectType()
class Usuario {
  @Field(() => ID) id: string;
  @Field() nombre: string;
  @Field() email: string;
  @Field() activo: boolean;
}

@Resolver(Usuario)
class UsuarioResolver {
  @Query(() => [Usuario])
  async usuarios(): Promise<Usuario[]> {
    return db.usuario.findMany();
  }

  @Query(() => Usuario, { nullable: true })
  async usuario(@Arg("id") id: string): Promise<Usuario | null> {
    return db.usuario.findUnique({ where: { id } });
  }

  @Mutation(() => Usuario)
  async crearUsuario(
    @Arg("nombre") nombre: string,
    @Arg("email") email: string,
  ): Promise<Usuario> {
    return db.usuario.create({ data: { nombre, email } });
  }
}

const schema = await buildSchema({
  resolvers: [UsuarioResolver],
  emitSchemaFile: true,
});

const server = new ApolloServer({ schema });
await server.start();
// Consulta GraphQL: { usuarios { id nombre email } }
// Mutacion: mutation { crearUsuario(nombre: "Andres", email: "a@test.com") { id } }
```

---

## 16.12 Idempotency Keys

```typescript
// Garantizar que POST/PUT se ejecutan una sola vez
app.post("/api/pagos", async (req, res) => {
  const idempotencyKey = req.headers["idempotency-key"] as string;
  if (!idempotencyKey) return res.status(400).json({ error: "Idempotency-Key requerido" });

  // Verificar si ya se proceso
  const existente = await cache.get(`idempotency:${idempotencyKey}`);
  if (existente) return res.status(200).json(existente);

  // Procesar pago
  const resultado = await procesarPago(req.body);

  // Guardar resultado (con TTL)
  await cache.set(`idempotency:${idempotencyKey}`, resultado, 86400);
  res.status(201).json(resultado);
});
```

---

## 16.13 Response Envelope (RFC 7807 Problem Details)

```typescript
// Estandarizar respuestas de error
interface ProblemDetails {
  type: string;
  title: string;
  status: number;
  detail: string;
  instance: string;
  errors?: Array<{ campo: string; mensaje: string }>;
}

app.use((err: Error, req: Request, res: Response, next: NextFunction) => {
  if (err instanceof ValidationError) {
    return res.status(400).json({
      type: "https://api.ejemplo.com/errors/validation",
      title: "Datos invalidos",
      status: 400,
      detail: err.message,
      instance: req.path,
      errors: err.errores,
    } satisfies ProblemDetails);
  }

  res.status(500).json({
    type: "https://api.ejemplo.com/errors/internal",
    title: "Error interno",
    status: 500,
    detail: "Error inesperado del servidor",
    instance: req.path,
  } satisfies ProblemDetails);
});
```

---

## 16.14 OpenTelemetry: Tracing Distribuido

```bash
pnpm add @opentelemetry/api @opentelemetry/sdk-node @opentelemetry/auto-instrumentations-node @opentelemetry/exporter-otlp-grpc
```

```typescript
// instrumentation.ts
import { NodeSDK } from "@opentelemetry/sdk-node";
import { OTLPTraceExporter } from "@opentelemetry/exporter-otlp-grpc";
import { getNodeAutoInstrumentations } from "@opentelemetry/auto-instrumentations-node";

const sdk = new NodeSDK({
  traceExporter: new OTLPTraceExporter({ url: "http://localhost:4317" }),
  instrumentations: [getNodeAutoInstrumentations()],
  serviceName: "api-usuarios",
});

sdk.start();
```

```typescript
// NestJS + OTel con NestInterceptor
import { Injectable, NestInterceptor, ExecutionContext, CallHandler } from "@nestjs/common";
import { trace, SpanStatusCode } from "@opentelemetry/api";

@Injectable()
export class TracingInterceptor implements NestInterceptor {
  intercept(context: ExecutionContext, next: CallHandler) {
    const req = context.switchToHttp().getRequest();
    const tracer = trace.getTracer("api-usuarios");

    return tracer.startActiveSpan(`${req.method} ${req.route.path}`, (span) => {
      span.setAttribute("http.method", req.method);
      span.setAttribute("http.url", req.url);

      return next.handle().pipe(
        tap({
          next: () => span.setStatus({ code: SpanStatusCode.OK }),
          error: (err) => {
            span.setStatus({ code: SpanStatusCode.ERROR, message: err.message });
            span.end();
          },
          finalize: () => span.end(),
        }),
      );
    });
  }
}

// Aplicar globalmente en NestJS
@Module({
  providers: [{ provide: APP_INTERCEPTOR, useClass: TracingInterceptor }],
})
export class AppModule {}
```

---

## 16.15 HTTP Caching

```typescript
// ETag y Cache-Control en NestJS
import { Controller, Get, Header, Res, Req } from "@nestjs/common";
import { Response, Request } from "express";

@Controller("usuarios")
export class UsuarioController {
  @Get(":id")
  async obtener(@Param("id") id: string, @Req() req: Request, @Res() res: Response) {
    const usuario = await this.service.obtenerPorId(id);
    const etag = crearETag(usuario);

    // Cache validation: si el cliente ya tiene la version actual
    if (req.headers["if-none-match"] === etag) {
      return res.status(304).end(); // Not Modified
    }

    res.set({
      "Cache-Control": "private, max-age=60",      // 1 min en navegador
      "ETag": etag,
      "Vary": "Accept-Encoding",
    });

    return res.json(usuario);
  }
}

function crearETag(datos: unknown): string {
  return crypto.createHash("md5").update(JSON.stringify(datos)).digest("hex");
}
```

---

## 16.16 Compression con Brotli

```typescript
// NestJS: comprimir respuestas
import compression from "compression";
import { NestFactory } from "@nestjs/core";
import zlib from "node:zlib";

const app = await NestFactory.create(AppModule);

app.use(compression({
  filter: (req, res) => {
    // No comprimir si ya esta comprimido (ej: imagenes)
    if (req.headers["content-type"]?.includes("image")) return false;
    return compression.filter(req, res);
  },
  level: zlib.constants.Z_BEST_COMPRESSION,
  threshold: 1024, // Solo comprimir respuestas > 1KB
}));

await app.listen(8080);
```

---

## 16.17 Connection Pooling y Prevencion de Agotamiento

```typescript
// Configuracion de pool de Prisma (CRITICO en produccion)
const prisma = new PrismaClient({
  datasources: {
    db: {
      url: process.env.DATABASE_URL,
    },
  },
  // Pool configuration
  connection: {
    pool: {
      max: 20,                 // Maximo conexiones simultaneas
      idle_timeout: 30_000,    // Cerrar despues de 30s idle
    },
  },
});
```

### Como saber el pool size optimo

```
pool_size = (num_cores * 2) + effective_spindle_count

Para Node.js (single-threaded, multiples workers):
  workers: 4 (cluster)
  pool por worker: 5
  pool total: 20

SIEMPRE menor que max_connections de PostgreSQL (default 100)

Si usas pgBouncer (RECOMENDADO en produccion):
  pool_size local = 20
  pgBouncer reserva solo ~40 conexiones reales a PostgreSQL
  para cientos de workers repartidos en multiples pods
```

### Sintomas de pool exhaustion

```typescript
// Error tipico: Timed out fetching a new connection from the connection pool
// Causas:
// 1. Transacciones largas (mala practica)
// 2. Conexiones no liberadas (olvidaste await, no cerraste)
// 3. pool_size demasiado pequeño para el trafico
// 4. Cuellos de botella en la BD (consultas lentas acumulan conexiones)

// Soluciones:
// 1. Poner timeout a las queries
await prisma.$queryRaw`SELECT ...`.catch(() => null);
// con timeout via Promise.race o statement_timeout en PostgreSQL

// 2. Monitorear metricas de pool
setInterval(() => {
  const metrics = prisma.$metrics.json();
  console.log({
    activas: metrics.pool.active,
    idle: metrics.pool.idle,
    esperando: metrics.pool.waiting,
  });
}, 10_000);

// 3. Circuit breaker para la BD (previene cascada)
const breaker = new CircuitBreaker(5, 10_000);
await breaker.ejecutar(() => prisma.usuario.findMany());
```

---

## 16.18 El Problema N+1 en ORMs

```typescript
// ❌ N+1: 1 query para usuarios + N queries para pedidos de cada usuario
const usuarios = await prisma.usuario.findMany();  // 1 query
for (const u of usuarios) {
  const pedidos = await prisma.pedido.findMany({   // N queries!
    where: { usuarioId: u.id },
  });
  console.log(u.nombre, pedidos.length);
}
// 100 usuarios = 101 queries a la BD 💀

// ✅ Solucion 1: include de Prisma (JOIN o batch query)
const usuariosConPedidos = await prisma.usuario.findMany({
  include: { pedidos: true },  // 1-2 queries total
});

// ✅ Solucion 2: DataLoader pattern (batching + caching)
import DataLoader from "dataloader";

const pedidosLoader = new DataLoader(async (usuarioIds: readonly string[]) => {
  const pedidos = await prisma.pedido.findMany({
    where: { usuarioId: { in: [...usuarioIds] } },  // 1 query para todos
  });

  // Agrupar por usuarioId para que DataLoader haga el lookup
  const porUsuario = new Map<string, typeof pedidos>();
  for (const p of pedidos) {
    const grupo = porUsuario.get(p.usuarioId) ?? [];
    grupo.push(p);
    porUsuario.set(p.usuarioId, grupo);
  }

  return usuarioIds.map((id) => porUsuario.get(id) ?? []);
});

// Uso: cada llamada individual se agrupa en una sola query
const pedidosDeAndres = await pedidosLoader.load("user-1");
```

### Detectar N+1 en desarrollo

```typescript
// Prisma: log de queries
const prisma = new PrismaClient({
  log: ["query"],
});
// Si ves 100 queries SELECT para 100 usuarios, tienes N+1

// NestJS: interceptor que detecta N+1
@Injectable()
export class NPlusOneInterceptor implements NestInterceptor {
  intercept(ctx: ExecutionContext, next: CallHandler) {
    const before = prisma.$metrics.json().pool.active;
    return next.handle().pipe(
      tap(() => {
        const after = prisma.$metrics.json().pool.active;
        if (after - before > 10) {
          logger.warn("Posible N+1 detectado", { queries: after - before });
        }
      }),
    );
  }
}
```

---

## 16.19 Distributed Rate Limiting (Redis-backed)

```typescript
// El rate limiter en memoria NO funciona con multiples instancias
// Solucion: Redis-backed sliding window

import { NestInterceptor, ExecutionContext, CallHandler, Injectable } from "@nestjs/common";
import { Redis } from "ioredis";

@Injectable()
export class DistributedRateLimitInterceptor implements NestInterceptor {
  constructor(private redis: Redis) {}

  async intercept(context: ExecutionContext, next: CallHandler) {
    const req = context.switchToHttp().getRequest();
    const key = `ratelimit:${req.ip}`;
    const now = Date.now();
    const windowMs = 60_000;   // 1 minuto
    const maxRequests = 100;

    // Sliding window en Redis
    const result = await this.redis
      .multi()
      .zremrangebyscore(key, 0, now - windowMs)  // Limpiar viejos
      .zcard(key)                                  // Contar actuales
      .zadd(key, now, `${now}-${Math.random()}`)   // Agregar este request
      .expire(key, Math.ceil(windowMs / 1000))     // TTL de la key
      .exec();

    const requestCount = result![1][1] as number;

    if (requestCount >= maxRequests) {
      throw new HttpException("Demasiadas peticiones", 429);
    }

    return next.handle();
  }
}
```

---

## 16.20 Idempotencia Real (At-Least-Once Processing)

```typescript
// Idempotencia: ejecutar la misma operacion N veces = mismo resultado que 1 vez
// Clave en sistemas distribuidos con reintentos

// NestJS: POST idempotente
@Post("pagos")
async crearPago(
  @Body() dto: CrearPagoDTO,
  @Headers("idempotency-key") idempotencyKey: string,
) {
  if (!idempotencyKey) {
    throw new BadRequestException("Idempotency-Key requerido");
  }

  // Verificar si ya procesamos esta key
  const existente = await redis.get(`idempotency:${idempotencyKey}`);
  if (existente) {
    return JSON.parse(existente); // Retornar resultado original
  }

  // Procesar pago
  const pago = await this.pagoService.procesar(dto);

  // Guardar resultado con TTL (24h)
  await redis.set(
    `idempotency:${idempotencyKey}`,
    JSON.stringify(pago),
    "EX",
    86400,
  );

  return pago;
}

// Webhook: idempotencia + verificacion de firma
@Post("webhooks/stripe")
async stripeWebhook(
  @Body() payload: Buffer,
  @Headers("stripe-signature") signature: string,
) {
  // 1. Verificar firma HMAC
  const event = this.stripe.webhooks.constructEvent(
    payload,
    signature,
    env.STRIPE_WEBHOOK_SECRET,
  );

  // 2. Idempotencia: el event.id de Stripe es unico
  const procesado = await redis.exists(`webhook:${event.id}`);
  if (procesado) return { received: true };

  // 3. Procesar evento
  await this.eventoService.procesar(event);

  // 4. Marcar como procesado
  await redis.set(`webhook:${event.id}`, "1", "EX", 86400 * 7);
  return { received: true };
}
```

---

## 16.21 API Versioning (Sunset y Deprecation)

```typescript
// NestJS: versionado de API
@Controller({ version: "2", path: "usuarios" })
export class UsuarioV2Controller {
  @Get(":id")
  obtenerV2(@Param("id") id: string) {
    return this.service.obtenerConFormatoNuevo(id);
  }
}

// Headers de deprecacion
@Controller({ version: "1", path: "usuarios" })
export class UsuarioV1Controller {
  @Get(":id")
  @Header("Deprecation", "true")
  @Header("Sunset", "Sat, 31 Dec 2025 23:59:59 GMT")
  @Header("Link", '</api/v2/usuarios>; rel="successor-version"')
  obtenerV1(@Param("id") id: string) {
    // Loggear para trackear quien sigue usando v1
    logger.warn("Uso de API deprecada", { version: "v1", id });
    return this.service.obtenerConFormatoViejo(id);
  }
}

// Monitorear versiones deprecadas
// Alerta si >5% del trafico usa versiones deprecadas
// Bloquear despues de la fecha Sunset
```

---

## Resumen del Capítulo

- Node.js tiene HTTP nativo, pero Express/Fastify ofrecen mejor DX y ecosistema.
- Express: maduro, gran ecosistema, middleware clasico. Fastify: mas rapido, schema validation nativa.
- Prisma: ORM type-safe con migraciones. Drizzle: SQL-like, ligero, sin codegen pesado.
- JWT para autenticacion stateless. bcrypt para hashing de passwords.
- Zod para validacion de entrada en todos los limites del sistema.
- Multer para file uploads con limites y filtros de tipo.
- Paginacion estandar con offset/limit y metadatos en la respuesta.
- Helmet para headers de seguridad. CORS configurado explicitamente.
- Siempre: validar entrada, sanitizar salida, rate limiting, y graceful shutdown.

En el siguiente y último capítulo encontraras ejercicios prácticos para consolidar lo aprendido.
