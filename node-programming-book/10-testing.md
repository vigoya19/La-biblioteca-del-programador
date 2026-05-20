# Capítulo 10: Testing

El testing en Node.js y TypeScript es un ecosistema rico: Vitest como runner moderno, Jest como alternativa consolidada, y Playwright para end-to-end. La clave esta en tests tipados, rapidos y mantenibles.

---

## 10.1 Vitest: Configuracion y Primeros Pasos

```bash
# Instalar Vitest
pnpm add -D vitest @vitest/ui @vitest/coverage-v8

# vitest.config.ts
import { defineConfig } from "vitest/config";
import path from "node:path";

export default defineConfig({
  test: {
    globals: true,          // No necesitas importar describe/it/expect
    environment: "node",
    include: ["src/**/*.test.ts", "tests/**/*.test.ts"],
    coverage: {
      provider: "v8",
      reporter: ["text", "json", "html"],
      include: ["src/**/*.ts"],
      exclude: ["src/**/*.test.ts", "src/**/*.d.ts"],
    },
  },
  resolve: {
    alias: {
      "@": path.resolve(__dirname, "./src"),
    },
  },
});
```

```typescript
// src/suma.ts
export function sumar(a: number, b: number): number {
  return a + b;
}

export function dividir(a: number, b: number): number {
  if (b === 0) {
    throw new Error("Division por cero");
  }
  return a / b;
}

// src/suma.test.ts
import { describe, it, expect } from "vitest";
import { sumar, dividir } from "./suma";

describe("sumar", () => {
  it("suma dos numeros positivos", () => {
    expect(sumar(3, 4)).toBe(7);
  });

  it("suma numeros negativos", () => {
    expect(sumar(-3, -4)).toBe(-7);
  });

  it("suma cero correctamente", () => {
    expect(sumar(5, 0)).toBe(5);
  });
});

describe("dividir", () => {
  it("divide dos numeros", () => {
    expect(dividir(10, 2)).toBe(5);
  });

  it("lanza error al dividir por cero", () => {
    expect(() => dividir(10, 0)).toThrow("Division por cero");
  });
});
```

```bash
# Ejecutar tests
pnpm test                    # vitest run (una vez)
pnpm test:watch              # vitest (watch mode)
pnpm test:coverage           # vitest run --coverage

# Filtrar tests
pnpm test -- sumar           # Solo tests que contengan "sumar"
pnpm test -- -t "division"   # Solo tests con nombre "division"
```

---

## 10.2 Tests de Integracion

```typescript
// src/modules/usuarios/usuario.repository.ts
import { PrismaClient } from "@prisma/client";

export class UsuarioRepository {
  constructor(private prisma: PrismaClient) {}

  async crear(datos: { nombre: string; email: string }) {
    return this.prisma.usuario.create({ data: datos });
  }

  async buscarPorEmail(email: string) {
    return this.prisma.usuario.findUnique({ where: { email } });
  }
}

// tests/integration/usuario.repository.test.ts
import { describe, it, expect, beforeAll, afterAll } from "vitest";
import { PrismaClient } from "@prisma/client";
import { UsuarioRepository } from "@/modules/usuarios/usuario.repository";

describe("UsuarioRepository (integracion)", () => {
  let prisma: PrismaClient;
  let repo: UsuarioRepository;

  beforeAll(async () => {
    prisma = new PrismaClient();
    repo = new UsuarioRepository(prisma);
    // Limpiar BD de prueba
    await prisma.usuario.deleteMany();
  });

  afterAll(async () => {
    await prisma.$disconnect();
  });

  it("crea un usuario correctamente", async () => {
    const usuario = await repo.crear({
      nombre: "Test User",
      email: "test@test.com",
    });

    expect(usuario.nombre).toBe("Test User");
    expect(usuario.email).toBe("test@test.com");
  });

  it("evita emails duplicados", async () => {
    await repo.crear({ nombre: "Uno", email: "dup@test.com" });

    await expect(
      repo.crear({ nombre: "Dos", email: "dup@test.com" }),
    ).rejects.toThrow();
  });
});
```

---

## 10.3 Mocks, Stubs y Spies

```typescript
import { describe, it, expect, vi } from "vitest";

// ─── Mock de modulo completo ───
vi.mock("@/shared/logger", () => ({
  logger: {
    info: vi.fn(),
    error: vi.fn(),
  },
}));

// ─── Mock de funcion con implementacion ───
const mockFetch = vi.fn();

describe("UsuarioService", () => {
  it("llama al repositorio con los datos correctos", async () => {
    // Spy: observar llamadas a una funcion real
    const repo = {
      crear: vi.fn().mockResolvedValue({ id: 1, nombre: "Test" }),
    };

    const servicio = new UsuarioService(repo);
    await servicio.registrar({ nombre: "Test", email: "test@test.com" });

    expect(repo.crear).toHaveBeenCalledTimes(1);
    expect(repo.crear).toHaveBeenCalledWith({
      nombre: "Test",
      email: "test@test.com",
    });
  });

  it("lanza error si el email ya existe", async () => {
    // Stub: simular un error especifico
    const repo = {
      buscarPorEmail: vi.fn().mockResolvedValue({ id: 2, email: "existe@test.com" }),
      crear: vi.fn(),
    };

    const servicio = new UsuarioService(repo);

    await expect(
      servicio.registrar({ nombre: "Otro", email: "existe@test.com" }),
    ).rejects.toThrow("Email ya registrado");
  });
});
```

### Fake: implementacion en memoria

```typescript
// Fake repository para tests (sin BD)
class UsuarioRepositoryFake implements UsuarioRepository {
  private usuarios: Usuario[] = [];
  private nextId = 1;

  async crear(datos: { nombre: string; email: string }): Promise<Usuario> {
    const usuario = { id: this.nextId++, ...datos };
    this.usuarios.push(usuario);
    return usuario;
  }

  async buscarPorEmail(email: string): Promise<Usuario | null> {
    return this.usuarios.find((u) => u.email === email) ?? null;
  }

  async buscarPorId(id: number): Promise<Usuario | null> {
    return this.usuarios.find((u) => u.id === id) ?? null;
  }
}

// Test usando el fake
describe("UsuarioService con fake", () => {
  it("crea usuario con repositorio en memoria", async () => {
    const repo = new UsuarioRepositoryFake();
    const servicio = new UsuarioService(repo);

    const usuario = await servicio.registrar({
      nombre: "Andres",
      email: "andres@test.com",
    });

    expect(usuario.nombre).toBe("Andres");
    expect(usuario.id).toBe(1);

    const encontrado = await repo.buscarPorEmail("andres@test.com");
    expect(encontrado).not.toBeNull();
  });
});
```

---

## 10.4 Snapshot Testing

```typescript
import { describe, it, expect } from "vitest";

describe("Formateador de usuarios", () => {
  it("formatea usuario correctamente", () => {
    const usuario = {
      id: 1,
      nombre: "Andres",
      email: "andres@ejemplo.com",
      creadoEn: new Date("2024-01-15T10:30:00Z"),
      activo: true,
      metadata: { plan: "pro", intentos: 3 },
    };

    const formateado = formatearUsuario(usuario);

    // Snapshot: guarda el resultado esperado en un archivo
    expect(formateado).toMatchSnapshot();
    // Primera ejecucion: crea __snapshots__/archivo.test.ts.snap
    // Siguientes ejecuciones: compara con el snapshot guardado
    // Si cambia intencionalmente: --update o -u
  });
});
```

---

## 10.5 End-to-End con Playwright

```bash
pnpm add -D @playwright/test
npx playwright install
```

```typescript
// tests/e2e/usuarios.spec.ts
import { test, expect } from "@playwright/test";

test.describe("API de usuarios", () => {
  test("GET /api/usuarios retorna lista", async ({ request }) => {
    const response = await request.get("http://localhost:3000/api/usuarios");
    expect(response.ok()).toBeTruthy();

    const body = await response.json();
    expect(Array.isArray(body)).toBe(true);
    expect(body.length).toBeGreaterThan(0);
    expect(body[0]).toHaveProperty("id");
    expect(body[0]).toHaveProperty("nombre");
  });

  test("POST /api/usuarios crea usuario", async ({ request }) => {
    const response = await request.post("http://localhost:3000/api/usuarios", {
      data: { nombre: "E2E User", email: "e2e@test.com" },
    });

    expect(response.status()).toBe(201);
    const body = await response.json();
    expect(body.nombre).toBe("E2E User");
  });

  test("GET /api/usuarios/:id con id inexistente retorna 404", async ({ request }) => {
    const response = await request.get("http://localhost:3000/api/usuarios/99999");
    expect(response.status()).toBe(404);
  });
});
```

---

## 10.6 Property-Based Testing con fast-check

```bash
pnpm add -D fast-check
```

```typescript
import { describe, it } from "vitest";
import * as fc from "fast-check";

function sumar(a: number, b: number): number {
  return a + b;
}

describe("sumar (property-based)", () => {
  it("es conmutativa: a + b = b + a", () => {
    fc.assert(
      fc.property(fc.integer(), fc.integer(), (a, b) => {
        expect(sumar(a, b)).toBe(sumar(b, a));
      }),
    );
  });

  it("es asociativa: (a + b) + c = a + (b + c)", () => {
    fc.assert(
      fc.property(fc.integer(), fc.integer(), fc.integer(), (a, b, c) => {
        expect(sumar(sumar(a, b), c)).toBe(sumar(a, sumar(b, c)));
      }),
    );
  });

  it("elemento neutro: a + 0 = a", () => {
    fc.assert(
      fc.property(fc.integer(), (a) => {
        expect(sumar(a, 0)).toBe(a);
      }),
    );
  });
});
```

---

---

## 10.8 Supertest: Testing HTTP Endpoints

```bash
pnpm add -D supertest @types/supertest
```

```typescript
import { describe, it, expect, beforeAll, afterAll } from "vitest";
import express from "express";
import request from "supertest";

const app = express();
app.use(express.json());

app.get("/api/health", (_req, res) => res.json({ status: "ok" }));

app.post("/api/usuarios", (req, res) => {
  const { nombre, email } = req.body;
  if (!nombre || !email) return res.status(400).json({ error: "Faltan campos" });
  res.status(201).json({ id: 1, nombre, email });
});

describe("API de usuarios", () => {
  it("GET /health retorna ok", async () => {
    const res = await request(app).get("/api/health");
    expect(res.status).toBe(200);
    expect(res.body).toEqual({ status: "ok" });
  });

  it("POST crea usuario", async () => {
    const res = await request(app)
      .post("/api/usuarios")
      .send({ nombre: "Andres", email: "a@test.com" })
      .expect(201);

    expect(res.body.nombre).toBe("Andres");
    expect(res.body.id).toBeDefined();
  });

  it("POST retorna 400 sin nombre", async () => {
    const res = await request(app)
      .post("/api/usuarios")
      .send({ email: "a@test.com" })
      .expect(400);

    expect(res.body.error).toBe("Faltan campos");
  });
});
```

---

## 10.9 Testcontainers: BD Real en Tests

```bash
pnpm add -D @testcontainers/postgresql testcontainers
```

```typescript
import { PostgreSqlContainer } from "@testcontainers/postgresql";
import { PrismaClient } from "@prisma/client";
import { describe, it, expect, beforeAll, afterAll } from "vitest";
import { execSync } from "node:child_process";

describe("Repositorio (integracion real)", () => {
  let prisma: PrismaClient;
  let container: PostgreSqlContainer;

  beforeAll(async () => {
    // Iniciar PostgreSQL en contenedor
    container = await new PostgreSqlContainer("postgres:16-alpine")
      .withDatabase("testdb")
      .withUsername("test")
      .withPassword("test")
      .start();

    // Conectar Prisma
    const url = container.getConnectionUri();
    process.env.DATABASE_URL = url;
    prisma = new PrismaClient({ datasources: { db: { url } } });

    // Ejecutar migraciones
    execSync("npx prisma migrate deploy", { env: { ...process.env, DATABASE_URL: url } });
  }, 30_000);

  afterAll(async () => {
    await prisma.$disconnect();
    await container.stop();
  });

  it("crea y recupera usuario", async () => {
    const creado = await prisma.usuario.create({
      data: { email: "test@test.com", nombre: "Test" },
    });
    expect(creado.nombre).toBe("Test");

    const encontrado = await prisma.usuario.findUnique({
      where: { id: creado.id },
    });
    expect(encontrado).not.toBeNull();
  });
});
```

---

## 10.10 Table-Driven Tests y Faker

```typescript
import { describe, it, expect } from "vitest";
import { faker } from "@faker-js/faker";

// Table-driven tests: array de casos
describe.each([
  { a: 1, b: 2, esperado: 3 },
  { a: -1, b: 1, esperado: 0 },
  { a: 0, b: 0, esperado: 0 },
  { a: 100, b: 200, esperado: 300 },
])("sumar($a, $b) = $esperado", ({ a, b, esperado }) => {
  it("resultado correcto", () => {
    expect(a + b).toBe(esperado);
  });
});

// Faker: datos realistas aleatorios
describe("UsuarioService", () => {
  it("crea usuario con datos validos", () => {
    const input = {
      nombre: faker.person.fullName(),
      email: faker.internet.email(),
      edad: faker.number.int({ min: 18, max: 99 }),
    };
    const usuario = servicio.crear(input);
    expect(usuario.email).toBe(input.email.toLowerCase());
  });
});

// Fake timers: controlar el tiempo
import { vi } from "vitest";

describe("Operaciones con tiempo", () => {
  it("expira despues del TTL", () => {
    vi.useFakeTimers();
    const cache = new Cache<string>(5000);
    cache.set("key", "value");

    vi.advanceTimersByTime(3000);
    expect(cache.get("key")).toBe("value"); // Aun no expira

    vi.advanceTimersByTime(3000);
    expect(cache.get("key")).toBeUndefined(); // Expirado

    vi.useRealTimers();
  });
});
```

---

## Buenas Practicas de Testing

### Checklist

```typescript
// 1. Nombra los tests describiendo el comportamiento esperado
// ❌ it("test1")
// ✅ it("lanza error cuando el email es invalido")

// 2. Sigue AAA: Arrange, Act, Assert
it("crea usuario cuando los datos son validos", async () => {
  // Arrange: preparar datos y dependencias
  const input = { nombre: "Andres", email: "andres@test.com" };
  const repo = new UsuarioRepositoryFake();
  const servicio = new UsuarioService(repo);

  // Act: ejecutar la accion
  const usuario = await servicio.registrar(input);

  // Assert: verificar el resultado
  expect(usuario.nombre).toBe("Andres");
  expect(usuario.id).toBeDefined();
});

// 3. Un concepto por test
// ❌ Test que prueba 5 cosas diferentes
// ✅ Cada test prueba un unico comportamiento

// 4. No testees implementacion, testea comportamiento
// ❌ expect(repo.crear).toHaveBeenCalled()
// ✅ expect(usuario.nombre).toBe("Andres")

// 5. Usa beforeEach para setup comun, no para compartir estado mutable
let repo: UsuarioRepositoryFake;

beforeEach(() => {
  repo = new UsuarioRepositoryFake(); // Nuevo para cada test
});

// 6. Tests deterministicos: mismo input = mismo output siempre
// ❌ Math.random(), new Date() sin mock
// ✅ Mockear o inyectar dependencias no deterministas
```

---

## Resumen del Capítulo

- **Vitest** es el runner moderno: compatible con Jest, mas rapido, soporte ESM nativo.
- Tests unitarios: importas la funcion, llamas, verificas con `expect().toBe()`.
- `vi.fn()` crea mocks/spies. `.mockResolvedValue()` para async.
- Fakes implementan la interfaz real con datos en memoria.
- Snapshot testing: util para objetos complejos que cambian poco.
- Playwright para E2E: testea la API real con peticiones HTTP.
- fast-check para property-based testing: genera inputs aleatorios y verifica propiedades.
- AAA (Arrange-Act-Assert) mantiene tests claros y mantenibles.
- No testees implementacion interna; testea comportamiento observable.

En el siguiente capitulo exploraremos los generics en TypeScript.

---

← [Capítulo anterior](09-modulos-paquetes.md) | [Inicio](README.md) | [Capítulo siguiente →](11-generics.md)
