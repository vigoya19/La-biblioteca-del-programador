# Capítulo 12: Buenas Prácticas

Escribir TypeScript y Node.js idiomatico va mas alla de conocer la sintaxis. Implica adoptar convenciones de la comunidad, configurar herramientas correctamente y seguir principios de diseño que escalan.

---

## 12.1 Convenciones de Nombrado

```typescript
// ✅ VARIABLES: camelCase, descriptivas
const contadorUsuarios = 0;
const maximoReintentos = 3;
const tiempoDeEspera = 5000;

// ❌ MAL: nombres genericos, abreviaturas oscuras
const c = 0;
const mx = 3;
const t = 5000;

// ✅ CONSTANTES: UPPER_SNAKE_CASE (valores inmutables globales)
const MAX_CONEXIONES = 100;
const API_BASE_URL = "https://api.ejemplo.com";
const TIMEOUT_POR_DEFECTO = 30_000;

// ✅ CLASES e INTERFACES: PascalCase
class UsuarioService {}
interface UsuarioRepository {}
type EstadoAsync<T> = {};

// ✅ ARCHIVOS: kebab-case
// usuario.service.ts
// usuario.repository.ts
// auth.middleware.ts

// ✅ FUNCIONES: verbo + sustantivo
function obtenerUsuarioPorId(id: string) {}
function validarEmail(email: string) {}
function formatearFecha(fecha: Date) {}

// ✅ BOOLEANOS: prefijo is/has/should
const isActivo = true;
const hasPermisos = false;
const shouldReintentar = true;
```

---

## 12.2 Organizacion de Codigo

### Estructura de un modulo (vertical slice)

```typescript
// src/modules/usuarios/usuario.service.ts
import type { Usuario, CrearUsuarioDTO } from "./usuario.types";
import type { UsuarioRepository } from "./usuario.repository";
import { NotFoundError, ConflictError } from "@/shared/errors";

export class UsuarioService {
  constructor(private readonly repo: UsuarioRepository) {}

  async crear(dto: CrearUsuarioDTO): Promise<Usuario> {
    const existente = await this.repo.buscarPorEmail(dto.email);
    if (existente) {
      throw new ConflictError("Email ya registrado");
    }
    return this.repo.crear(dto);
  }

  async obtenerPorId(id: string): Promise<Usuario> {
    const usuario = await this.repo.buscarPorId(id);
    if (!usuario) {
      throw new NotFoundError("Usuario", id);
    }
    return usuario;
  }
}
```

### Principio de proximidad

```typescript
// ✅ BIEN: codigo relacionado junto
const PUERTO_POR_DEFECTO = 8080;
const HOST_POR_DEFECTO = "localhost";

interface ConfigServidor {
  puerto: number;
  host: string;
}

function crearConfig(params?: Partial<ConfigServidor>): ConfigServidor {
  return {
    puerto: params?.puerto ?? PUERTO_POR_DEFECTO,
    host: params?.host ?? HOST_POR_DEFECTO,
  };
}
```

### Barrel exports (index.ts)

```typescript
// src/modules/usuarios/index.ts
export { UsuarioService } from "./usuario.service";
export { UsuarioRepository } from "./usuario.repository";
export type { Usuario, CrearUsuarioDTO } from "./usuario.types";
export { usuarioRoutes } from "./usuario.routes";

// Consumo limpio:
import { UsuarioService, type Usuario } from "@/modules/usuarios";
```

---

## 12.3 tsconfig.json Optimo

```json
{
  "compilerOptions": {
    // ─── Modulo y salida ───
    "target": "ES2022",
    "module": "NodeNext",
    "moduleResolution": "NodeNext",
    "outDir": "./dist",
    "rootDir": "./src",

    // ─── Strict (TODO deberia estar activado) ───
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "noImplicitReturns": true,
    "noFallthroughCasesInSwitch": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true,
    "exactOptionalPropertyTypes": false,

    // ─── Interoperabilidad ───
    "esModuleInterop": true,
    "allowSyntheticDefaultImports": true,
    "resolveJsonModule": true,
    "isolatedModules": true,

    // ─── Emision ───
    "declaration": true,
    "declarationMap": true,
    "sourceMap": true,
    "removeComments": true,

    // ─── Path aliases ───
    "baseUrl": ".",
    "paths": {
      "@/*": ["./src/*"]
    },

    // ─── Rendimiento ───
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true
  },
  "include": ["src/**/*.ts"],
  "exclude": ["node_modules", "dist", "tests"]
}
```

| Opcion | Por que activarla |
|--------|-------------------|
| `strict: true` | Habilita todas las verificaciones estrictas |
| `noUncheckedIndexedAccess` | Arrays y objetos devuelven `T \| undefined` |
| `noImplicitReturns` | Obliga a retornar en todas las ramas |
| `noUnusedLocals` | Detecta variables no usadas |
| `noUnusedParameters` | Detecta parametros no usados |
| `skipLibCheck` | Acelera compilacion (no verifica `.d.ts`) |

---

## 12.4 Principios SOLID en TypeScript

```typescript
// S: Single Responsibility
// ❌ Clase que hace demasiado
class ReporteService {
  generarDatos() {}
  formatearPDF() {}
  enviarEmail() {}
  guardarEnDisco() {}
}

// ✅ Responsabilidades separadas
class GeneradorDatos {}
class FormateadorPDF {}
class EnviadorEmail {}

// O: Open/Closed (abierto a extension, cerrado a modificacion)
interface Descuento {
  aplicar(precio: number): number;
}

class DescuentoVIP implements Descuento {
  aplicar(precio: number): number { return precio * 0.8; }
}

// Nuevo descuento sin modificar el existente
class DescuentoBlackFriday implements Descuento {
  aplicar(precio: number): number { return precio * 0.5; }
}

// L: Liskov Substitution
// Las subclases deben ser sustituibles por sus clases base

// I: Interface Segregation
// ❌ Interface enorme
interface Worker {
  trabajar(): void;
  comer(): void;
  dormir(): void;
  reunirse(): void;
}

// ✅ Interfaces segregadas
interface Trabajable { trabajar(): void; }
interface Descansable { comer(): void; dormir(): void; }

// D: Dependency Inversion
// Depende de abstracciones, no de implementaciones
class Servicio {
  constructor(private repo: UsuarioRepository) {} // Interface, no clase concreta
}
```

---

## 12.5 Code Review Checklist

| # | Que verificar | Detalle |
|---|---------------|---------|
| 1 | **Tipos estrictos** | Sin `any` (salvo excepciones documentadas) |
| 2 | **Errores manejados** | Nunca promesas sin `.catch()` o `try/catch` |
| 3 | **Validacion de entrada** | Zod/Yup en limites del sistema (handlers, controllers) |
| 4 | **Dependencias explicitas** | Inyectadas via constructor (no `new` dentro de clases) |
| 5 | **Funciones pequeñas** | Ideal < 20 lineas; maximo ~50 lineas |
| 6 | **Nombres descriptivos** | Sin abreviaturas, sin nombres genericos (`data`, `item`, `val`) |
| 7 | **Sin mutacion inesperada** | Preferir spread/immutabilidad; evitar `push` a arrays pasados por parametro |
| 8 | **async/await sobre .then()** | Mas legible y facil de depurar |
| 9 | **Logging en catch** | Siempre loggear errores con contexto suficiente |
| 10 | **Sin console.log en produccion** | Usar logger (pino, winston) |
| 11 | **Tests para nueva funcionalidad** | Unitarios + al menos un test de integracion |
| 12 | **Sin secretos en codigo** | Usar variables de entorno (nunca hardcodeados) |
| 13 | **Imports organizados** | Node builtins → externos → internos; sin imports no usados |
| 14 | **Tipos exportados** | Interfaces/tipos publicos exportados desde barrel files |
| 15 | **Manejo de edge cases** | null, undefined, arrays vacios, strings vacios, timeouts |

---

## 12.6 Configuracion de ESLint/Biome

```json
// biome.json
{
  "$schema": "https://biomejs.dev/schemas/1.9.4/schema.json",
  "organizeImports": { "enabled": true },
  "formatter": {
    "indentStyle": "space",
    "indentWidth": 2,
    "lineWidth": 100,
    "lineEnding": "lf"
  },
  "linter": {
    "enabled": true,
    "rules": {
      "recommended": true,
      "correctness": {
        "noUnusedVariables": "error",
        "noUnusedImports": "error",
        "useExhaustiveDependencies": "warn"
      },
      "suspicious": {
        "noExplicitAny": "error",
        "noArrayIndexKey": "warn"
      },
      "style": {
        "useConst": "error",
        "useTemplate": "error",
        "noNegationElse": "error"
      },
      "complexity": {
        "noBannedTypes": "error",
        "useLiteralKeys": "error"
      }
    }
  },
  "javascript": {
    "formatter": {
      "semicolons": "always",
      "quoteStyle": "double",
      "trailingCommas": "all",
      "arrowParentheses": "always"
    }
  }
}
```

---

## 12.7 Variables de Entorno y Configuracion

```typescript
// src/config/env.ts
import { z } from "zod";

const envSchema = z.object({
  NODE_ENV: z.enum(["development", "production", "test"]).default("development"),
  PORT: z.coerce.number().int().positive().default(8080),
  DATABASE_URL: z.string().url(),
  REDIS_URL: z.string().url().optional(),
  JWT_SECRET: z.string().min(32),
  LOG_LEVEL: z.enum(["debug", "info", "warn", "error"]).default("info"),
});

export type Env = z.infer<typeof envSchema>;

// Validar en inicio (el proceso falla si falta algo)
export const env: Env = envSchema.parse(process.env);
```

---

## 12.8 Graceful Shutdown

```typescript
// src/app.ts
import { createServer } from "node:http";

async function iniciarServidor() {
  const server = createServer(/* ... */);

  // Señales de interrupcion
  const shutdown = async (signal: string) => {
    console.log(`\nRecibida señal ${signal}. Cerrando...`);
    server.close(() => {
      console.log("Servidor cerrado");
      process.exit(0);
    });

    // Forzar cierre despues de 10s
    setTimeout(() => {
      console.error("Forzando cierre despues de timeout");
      process.exit(1);
    }, 10_000);
  };

  process.on("SIGTERM", () => shutdown("SIGTERM"));
  process.on("SIGINT", () => shutdown("SIGINT"));

  server.listen(env.PORT, () => {
    console.log(`Servidor en http://localhost:${env.PORT}`);
  });
}
```

---

## 12.9 Docker Multi-Stage para Node.js + TypeScript

```dockerfile
# Dockerfile
# ─── Stage 1: Instalar dependencias ───
FROM node:22-alpine AS deps
WORKDIR /app
COPY package.json pnpm-lock.yaml ./
RUN corepack enable && pnpm install --frozen-lockfile --prod

# ─── Stage 2: Compilar TypeScript ───
FROM node:22-alpine AS build
WORKDIR /app
COPY package.json pnpm-lock.yaml tsconfig.json ./
RUN corepack enable && pnpm install --frozen-lockfile
COPY src/ ./src/
RUN pnpm build

# ─── Stage 3: Imagen final (solo prod) ───
FROM node:22-alpine AS runner
WORKDIR /app
RUN addgroup --system --gid 1001 nodejs && \
    adduser --system --uid 1001 appuser

COPY --from=deps /app/node_modules ./node_modules
COPY --from=build /app/dist ./dist
COPY --from=build /app/package.json ./

USER appuser
EXPOSE 8080
ENV NODE_ENV=production
CMD ["node", "dist/index.js"]
```

```yaml
# .github/workflows/ci.yml
name: CI

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16-alpine
        env:
          POSTGRES_USER: test
          POSTGRES_PASSWORD: test
          POSTGRES_DB: testdb
        ports: ["5432:5432"]

    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v4
      - uses: actions/setup-node@v4
        with: { node-version: 22, cache: "pnpm" }

      - run: pnpm install --frozen-lockfile
      - run: pnpm typecheck
      - run: pnpm lint
      - run: pnpm test --coverage
        env: { DATABASE_URL: "postgresql://test:test@localhost:5432/testdb" }
```

---

## 12.10 Git Hooks con Husky + Commitlint

```bash
pnpm add -D husky lint-staged @commitlint/cli @commitlint/config-conventional
pnpm exec husky init
```

```bash
# .husky/commit-msg
pnpm exec commitlint --edit $1

# .husky/pre-commit
pnpm exec lint-staged
```

```json
// commitlint.config.js
export default { extends: ["@commitlint/config-conventional"] };
// Commits: feat: agregar login, fix: corregir timeout, chore: actualizar deps
```

```json
// package.json
{
  "lint-staged": {
    "*.ts": ["biome format --write", "biome lint --apply"],
    "*.{json,md,yaml}": ["biome format --write"]
  }
}
```

---

## 12.11 DI Containers (TSyringe y Awilix)

```typescript
// TSyringe: decorators para inyeccion de dependencias
import "reflect-metadata";
import { injectable, inject, container } from "tsyringe";

@injectable()
class UsuarioRepository {
  constructor(@inject("Database") private db: Database) {}
  async buscar(id: string) { return this.db.query(...); }
}

@injectable()
class UsuarioService {
  constructor(private repo: UsuarioRepository) {}
  async obtenerPerfil(id: string) { return this.repo.buscar(id); }
}

// Registrar y resolver
container.register("Database", { useValue: new Database(...) });
const service = container.resolve(UsuarioService);
```

```typescript
// Awilix: sin decorators, functional DI
import { createContainer, asClass, asValue, asFunction } from "awilix";

const container = createContainer();

container.register({
  db: asValue(new Database(process.env.DATABASE_URL!)),
  usuarioRepo: asClass(UsuarioRepository),
  usuarioService: asClass(UsuarioService),
});

const service = container.resolve<UsuarioService>("usuarioService");
```

---

## 12.12 Pino Logger (Alternativa de Alto Rendimiento)

```typescript
import pino from "pino";

const logger = pino({
  level: process.env.LOG_LEVEL ?? "info",
  transport: process.env.NODE_ENV === "development"
    ? { target: "pino-pretty", options: { colorize: true } }
    : undefined,
  serializers: {
    req: pino.stdSerializers.req,
    res: pino.stdSerializers.res,
    err: pino.stdSerializers.err,
  },
  redact: ["req.headers.authorization", "req.headers.cookie"],
});

logger.info({ puerto: 8080 }, "Servidor iniciado");
logger.error({ err: new Error("DB timeout"), query: "SELECT..." }, "Error BD");
logger.child({ requestId: "abc" }).info("Procesando peticion");
```

---

## 12.13 The Twelve-Factor App en Node.js

Metodologia para construir aplicaciones cloud-native. Los 12 factores:

### I. Codebase (Un repositorio, multiples deploys)

```
mi-app/
├── .github/workflows/     # CI/CD
├── docker-compose.yml      # Dev
├── k8s/                    # Produccion
│   ├── staging/
│   └── production/
└── src/
```

### III. Config (Variables de entorno, nunca en codigo)

```typescript
// ✅ Configuracion validada al inicio
import { z } from "zod";

const envSchema = z.object({
  DATABASE_URL: z.string().url(),
  REDIS_URL: z.string().url(),
  JWT_SECRET: z.string().min(32),
  PORT: z.coerce.number().default(8080),
});

export const env = envSchema.parse(process.env); // Falla rapido si falta algo

// ❌ Nunca: if (process.env.NODE_ENV === "production") { ... }
// Usa configuracion explicita, no condicionales de entorno
```

### VI. Processes (Stateless, share-nothing)

```typescript
// ❌ MAL: estado en memoria que se pierde al reiniciar
const sesiones = new Map(); // Se pierde en crash/deploy

// ✅ BIEN: estado externalizado
const sesiones = await redis.get(`session:${sessionId}`);
```

### IX. Disposability (Arranque rapido, apagado elegante)

```typescript
// Ver seccion 12.14 Graceful Shutdown
```

### X. Dev/Prod parity (Mismos servicios en todos lados)

```yaml
# docker-compose.yml para desarrollo
services:
  postgres:
    image: postgres:16-alpine  # Misma version que produccion
  redis:
    image: redis:7-alpine
```

### XI. Logs (Streams, no archivos)

```typescript
// ✅ Logs a stdout (la plataforma los captura)
const logger = pino(); // Escribe a stdout por defecto

// ❌ NUNCA escribir logs a archivo en el contenedor
// fs.appendFileSync("/var/log/app.log", ...)
```

---

## 12.14 Health Checks, Readiness y Liveness

```typescript
// NestJS: HealthController para Kubernetes
import { Controller, Get } from "@nestjs/common";
import { HealthCheckService, PrismaHealthIndicator, MemoryHealthIndicator } from "@nestjs/terminus";

@Controller("health")
export class HealthController {
  constructor(
    private health: HealthCheckService,
    private prisma: PrismaHealthIndicator,
    private memory: MemoryHealthIndicator,
  ) {}

  @Get()
  readiness() {
    return this.health.check([
      // Liveness: ¿está vivo el proceso?
      async () => ({ server: { status: "up" } }),
      // Readiness: ¿puede recibir tráfico?
      async () => this.prisma.pingCheck("database", this.prismaService),
      // Memoria: ¿no estamos en OOM?
      async () => this.memory.checkHeap("memory_heap", 200 * 1024 * 1024),
    ]);
  }

  @Get("live")
  liveness() {
    return { status: "ok", uptime: process.uptime() };
  }
}
```

### Graceful Shutdown Real (NestJS)

```typescript
// main.ts
import { NestFactory } from "@nestjs/core";
import { AppModule } from "./app.module";

async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  const server = app.getHttpServer();

  await app.listen(env.PORT);
  console.log(`Servidor listo en :${env.PORT}`);

  // Señal de readiness: el proceso puede recibir tráfico
  process.send?.("ready");

  // Keep-alive: cerrar conexiones idle antes del shutdown
  server.keepAliveTimeout = 61_000;    // > load balancer timeout
  server.headersTimeout = 65_000;       // > keepAliveTimeout

  const shutdown = async (signal: string) => {
    console.log(`Señal ${signal} recibida. Iniciando shutdown...`);

    // 1. Dejar de aceptar nuevas peticiones (readiness falla)
    // El health check ya retorna 503 porque paramos readiness

    // 2. Esperar que las peticiones en vuelo terminen (max 25s)
    await app.close(); // NestJS cierra conexiones y resuelve pending requests

    // 3. Cerrar conexiones de BD, Redis, etc.
    await prisma.$disconnect();
    await redis.quit();

    console.log("Shutdown completo");
    process.exit(0);
  };

  process.on("SIGTERM", () => shutdown("SIGTERM"));
  process.on("SIGINT", () => shutdown("SIGINT"));

  // Forzar salida si no terminamos en 30s
  setTimeout(() => {
    console.error("Shutdown forzado por timeout");
    process.exit(1);
  }, 30_000).unref();
}
bootstrap();
```

### Drenar conexiones antes del shutdown

```typescript
// En Kubernetes, el pod recibe SIGTERM y:
// 1. El health check de readiness falla -> el Service deja de enviar tráfico
// 2. Esperamos terminationGracePeriodSeconds (ej: 30s)
// 3. Drenamos conexiones activas
// 4. Salimos limpiamente

// Configurar en tu deployment:
// terminationGracePeriodSeconds: 45  (mayor que el tiempo de shutdown)
// Y en el health check, readiness debe fallar ANTES de recibir SIGTERM
```

---

## 12.15 Secrets Management

```typescript
// ❌ NUNCA: secretos en .env commiteado
// .env:
// DATABASE_URL=postgres://user:password@host/db  <- JAMAS COMMITEAR

// ✅ .env.example (template SIN valores reales)
// DATABASE_URL=postgres://user:password@host:5432/db

// ✅ Estrategia 1: Secretos en Vault / Infisical / AWS Secrets Manager
import { SecretsManager } from "@aws-sdk/client-secrets-manager";

async function cargarSecrets(): Promise<Record<string, string>> {
  if (process.env.NODE_ENV === "development") {
    // Local: cargar de .env (gitignored)
    return (await import("dotenv")).config().parsed!;
  }

  // Produccion: rotar desde Secrets Manager
  const client = new SecretsManager();
  const secret = await client.getSecretValue({ SecretId: "prod/api-secrets" });
  return JSON.parse(secret.SecretString!);
}

// ✅ Estrategia 2: Nunca exponer secretos en process.env global
// Extraer al inicio y eliminar
const config = {
  dbUrl: process.env.DATABASE_URL!,
  jwtSecret: process.env.JWT_SECRET!,
};
delete process.env.DATABASE_URL;
delete process.env.JWT_SECRET;
// Ahora un console.log o dump de errores no filtrará secretos

// ✅ Estrategia 3: Rotar secretos sin reiniciar
// Escuchar cambios en Secrets Manager y actualizar config
setInterval(async () => {
  const nuevosSecretos = await cargarSecrets();
  config.dbUrl = nuevosSecretos.DATABASE_URL;
  // Reconectar con las nuevas credenciales
}, 300_000); // Cada 5 min
```

---

## 12.16 Supply Chain Security

```bash
# Auditar vulnerabilidades
npm audit
pnpm audit
pnpm audit --audit-level=high

# npm audit signatures (integridad del registry)
npm config set audit-signatures true

# Instalacion estricta (solo lockfile, no resuelve de nuevo)
npm ci          # Respeta package-lock.json exactamente
pnpm install --frozen-lockfile

# Generar SBOM (Software Bill of Materials)
npx @cyclonedx/cyclonedx-npm --output-file sbom.json

# Escanear dependencias con Socket.dev
npx socket scan
```

```yaml
# CI: Verificar seguridad
security:
  steps:
    - run: pnpm audit --audit-level=high
    - run: npx socket scan
    - run: npx @cyclonedx/cyclonedx-npm --sbom-format json
```

---

## Resumen del Capítulo

- Nombra variables en camelCase, clases/interfaces en PascalCase, archivos en kebab-case.
- Usa barrel exports (`index.ts`) para simplificar imports.
- `strict: true` en tsconfig es obligatorio. `noUncheckedIndexedAccess` previene bugs.
- Aplica SOLID: responsabilidad unica, interfaces segregadas, dependencia invertida.
- La checklist de code review (15 puntos) detecta problemas comunes sistematicamente.
- Biome proporciona linting y formateo rapido como alternativa moderna a ESLint + Prettier.
- Valida variables de entorno con Zod al iniciar. El proceso debe fallar si falta configuracion.
- Implementa graceful shutdown capturando SIGTERM y SIGINT.

En el siguiente capítulo exploraremos los patrones de diseño aplicados a Node.js.

---

← [Capítulo anterior](11-generics.md) | [Inicio](README.md) | [Capítulo siguiente →](13-patrones-de-diseno.md)
