# Capítulo 14: Arquitectura Hexagonal y Domain-Driven Design

La arquitectura hexagonal (Ports & Adapters) combinada con Domain-Driven Design produce aplicaciones Node.js mantenibles, testeables e independientes de la infraestructura. TypeScript permite expresar el dominio con tipos precisos.

> [!NOTE]
> Para comprender en profundidad los fundamentos teóricos detrás de estos patrones, te recomendamos leer el [Capítulo 6: Domain-Driven Design (DDD)](../software-architecture-programming-book/06-ddd.md) y el [Capítulo 7: Arquitectura Hexagonal y Clean Architecture](../software-architecture-programming-book/07-hexagonal-clean.md) del libro de **Arquitectura de Software** en tu workspace.

---

## 14.1 Principios de la Arquitectura Hexagonal

```
         ┌──────────────────────────────────┐
         │        ADAPTADORES               │
         │  ┌────────────────────────────┐  │
  HTTP ──┼──┤     CAPA DE APLICACION     ├──┼── PostgreSQL
         │  │   ┌──────────────────┐     │  │
  gRPC ──┼──┤   │   DOMINIO PURO   │     ├──┼── Redis
         │  │   └──────────────────┘     │  │
  CLI  ──┼──┤                            ├──┼── RabbitMQ
         │  └────────────────────────────┘  │
         │        ADAPTADORES               │
         └──────────────────────────────────┘
```

**Regla fundamental**: El dominio NO conoce nada del exterior. Las dependencias apuntan hacia adentro.

---

## 14.2 Estructura del Proyecto

```
mi-api/
├── src/
│   ├── index.ts                    # Composicion raiz
│   ├── domain/
│   │   ├── usuario/
│   │   │   ├── usuario.entity.ts   # Entidad
│   │   │   ├── email.value-object.ts
│   │   │   └── usuario.errors.ts
│   │   └── orden/
│   │       └── orden.aggregate.ts
│   ├── application/
│   │   ├── usuario/
│   │   │   ├── registrar.usecase.ts
│   │   │   └── obtener-perfil.usecase.ts
│   │   └── ports/
│   │       ├── usuario.repository.ts   # Interface (puerto)
│   │       └── email.service.ts        # Interface (puerto)
│   └── infrastructure/
│       ├── persistence/
│       │   └── usuario.prisma.repository.ts  # Adaptador
│       ├── email/
│       │   └── nodemailer.service.ts         # Adaptador
│       └── http/
│           ├── usuario.controller.ts         # Adaptador
│           └── usuario.routes.ts
```

---

## 14.3 Capa de Dominio

### Value Objects

```typescript
// src/domain/usuario/email.value-object.ts
export class Email {
  private constructor(private readonly value: string) {}

  static crear(email: string): Email {
    const normalizado = email.trim().toLowerCase();
    if (!Email.esValido(normalizado)) {
      throw new Error(`Email invalido: ${email}`);
    }
    return new Email(normalizado);
  }

  static esValido(email: string): boolean {
    return /^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(email);
  }

  toString(): string {
    return this.value;
  }

  equals(other: Email): boolean {
    return this.value === other.value;
  }
}
```

### Entidad

```typescript
// src/domain/usuario/usuario.entity.ts
import { Email } from "./email.value-object";

export interface UsuarioProps {
  id: string;
  email: Email;
  nombre: string;
  activo: boolean;
  creadoEn: Date;
}

export class Usuario {
  private constructor(private readonly props: UsuarioProps) {}

  static crear(props: { id: string; email: Email; nombre: string }): Usuario {
    if (!props.nombre.trim()) {
      throw new Error("Nombre no puede estar vacio");
    }
    return new Usuario({
      id: props.id,
      email: props.email,
      nombre: props.nombre.trim(),
      activo: true,
      creadoEn: new Date(),
    });
  }

  // Getters (no setters: la entidad controla su estado)
  get id(): string { return this.props.id; }
  get email(): Email { return this.props.email; }
  get nombre(): string { return this.props.nombre; }
  get activo(): boolean { return this.props.activo; }

  // Metodos de dominio (comportamiento)
  cambiarNombre(nuevo: string): void {
    if (!nuevo.trim()) throw new Error("Nombre no puede estar vacio");
    this.props.nombre = nuevo.trim();
  }

  desactivar(): void {
    this.props.activo = false;
  }

  toPlainObject(): UsuarioProps {
    return { ...this.props };
  }
}
```

---

## 14.4 Capa de Aplicacion (Puertos y Casos de Uso)

### Puertos (interfaces)

```typescript
// src/application/ports/usuario.repository.ts
import { Usuario } from "@/domain/usuario/usuario.entity";
import { Email } from "@/domain/usuario/email.value-object";

export interface UsuarioRepository {
  guardar(usuario: Usuario): Promise<void>;
  buscarPorId(id: string): Promise<Usuario | null>;
  buscarPorEmail(email: Email): Promise<Usuario | null>;
}
```

### Caso de uso

```typescript
// src/application/usuario/registrar.usecase.ts
import { UsuarioRepository } from "../ports/usuario.repository";
import { Usuario } from "@/domain/usuario/usuario.entity";
import { Email } from "@/domain/usuario/email.value-object";
import { randomUUID } from "node:crypto";

// DTO de entrada
export interface RegistrarUsuarioInput {
  email: string;
  nombre: string;
}

// DTO de salida
export interface RegistrarUsuarioOutput {
  id: string;
  email: string;
  nombre: string;
}

export class RegistrarUsuarioUseCase {
  constructor(private readonly repo: UsuarioRepository) {}

  async ejecutar(input: RegistrarUsuarioInput): Promise<RegistrarUsuarioOutput> {
    // Validar (puede lanzar error de dominio)
    const email = Email.crear(input.email);

    // Verificar unicidad
    const existente = await this.repo.buscarPorEmail(email);
    if (existente) {
      throw new Error("Email ya registrado");
    }

    // Crear entidad de dominio
    const usuario = Usuario.crear({
      id: randomUUID(),
      email,
      nombre: input.nombre,
    });

    // Persistir
    await this.repo.guardar(usuario);

    // Retornar DTO
    return {
      id: usuario.id,
      email: usuario.email.toString(),
      nombre: usuario.nombre,
    };
  }
}
```

---

## 14.5 Capa de Infraestructura (Adaptadores)

### Adaptador de persistencia con Prisma

```typescript
// src/infrastructure/persistence/usuario.prisma.repository.ts
import { PrismaClient } from "@prisma/client";
import { UsuarioRepository } from "@/application/ports/usuario.repository";
import { Usuario } from "@/domain/usuario/usuario.entity";
import { Email } from "@/domain/usuario/email.value-object";

export class UsuarioPrismaRepository implements UsuarioRepository {
  constructor(private readonly prisma: PrismaClient) {}

  async guardar(usuario: Usuario): Promise<void> {
    const datos = usuario.toPlainObject();
    await this.prisma.usuario.upsert({
      where: { id: datos.id },
      create: {
        id: datos.id,
        email: datos.email.toString(),
        nombre: datos.nombre,
        activo: datos.activo,
      },
      update: {
        nombre: datos.nombre,
        activo: datos.activo,
      },
    });
  }

  async buscarPorId(id: string): Promise<Usuario | null> {
    const row = await this.prisma.usuario.findUnique({ where: { id } });
    if (!row) return null;
    return Usuario.crear({
      id: row.id,
      email: Email.crear(row.email),
      nombre: row.nombre,
    });
  }

  async buscarPorEmail(email: Email): Promise<Usuario | null> {
    const row = await this.prisma.usuario.findUnique({
      where: { email: email.toString() },
    });
    if (!row) return null;
    return Usuario.crear({
      id: row.id,
      email: Email.crear(row.email),
      nombre: row.nombre,
    });
  }
}
```

### Adaptador HTTP

```typescript
// src/infrastructure/http/usuario.controller.ts
import { Router, Request, Response } from "express";
import { RegistrarUsuarioUseCase } from "@/application/usuario/registrar.usecase";
import { z } from "zod";

const registrarSchema = z.object({
  email: z.string().email(),
  nombre: z.string().min(1).max(100),
});

export function crearUsuarioRouter(useCase: RegistrarUsuarioUseCase): Router {
  const router = Router();

  router.post("/usuarios", async (req: Request, res: Response) => {
    try {
      const input = registrarSchema.parse(req.body);
      const output = await useCase.ejecutar(input);
      res.status(201).json(output);
    } catch (error) {
      if (error instanceof z.ZodError) {
        res.status(400).json({ error: "Datos invalidos", detalles: error.errors });
      } else if (error instanceof Error && error.message === "Email ya registrado") {
        res.status(409).json({ error: error.message });
      } else {
        res.status(500).json({ error: "Error interno" });
      }
    }
  });

  return router;
}
```

---

## 14.6 Composicion Raiz (index.ts)

```typescript
// src/index.ts
import express from "express";
import { PrismaClient } from "@prisma/client";
import { RegistrarUsuarioUseCase } from "@/application/usuario/registrar.usecase";
import { UsuarioPrismaRepository } from "@/infrastructure/persistence/usuario.prisma.repository";
import { crearUsuarioRouter } from "@/infrastructure/http/usuario.controller";

async function main() {
  const prisma = new PrismaClient();
  const app = express();
  app.use(express.json());

  // Ensamblar dependencias
  const usuarioRepo = new UsuarioPrismaRepository(prisma);
  const registrarUseCase = new RegistrarUsuarioUseCase(usuarioRepo);
  const usuarioRouter = crearUsuarioRouter(registrarUseCase);

  app.use("/api", usuarioRouter);

  app.listen(8080, () => console.log("Servidor en http://localhost:8080"));
}

main().catch(console.error);
```

---

## 14.7 Testing en Arquitectura Hexagonal

```typescript
// Test del dominio (sin dependencias externas)
import { describe, it, expect } from "vitest";
import { Email } from "@/domain/usuario/email.value-object";
import { Usuario } from "@/domain/usuario/usuario.entity";

describe("Usuario (dominio)", () => {
  it("crea usuario valido", () => {
    const email = Email.crear("andres@test.com");
    const usuario = Usuario.crear({ id: "1", email, nombre: "Andres" });

    expect(usuario.nombre).toBe("Andres");
    expect(usuario.email.toString()).toBe("andres@test.com");
  });

  it("rechaza email invalido", () => {
    expect(() => Email.crear("invalido")).toThrow();
  });
});

// Test del caso de uso (con mock del repositorio)
describe("RegistrarUsuarioUseCase", () => {
  it("registra usuario con email nuevo", async () => {
    const repo = {
      buscarPorEmail: vi.fn().mockResolvedValue(null),
      guardar: vi.fn().mockResolvedValue(undefined),
    };

    const useCase = new RegistrarUsuarioUseCase(repo);
    const output = await useCase.ejecutar({
      email: "nuevo@test.com",
      nombre: "Nuevo",
    });

    expect(output.email).toBe("nuevo@test.com");
    expect(repo.guardar).toHaveBeenCalled();
  });
});
```

---

## 14.8 Aggregate Roots

Un Aggregate es un grupo de entidades que se tratan como una unidad. El Aggregate Root es la entidad "principal" que garantiza la consistencia del grupo.

```typescript
// src/domain/orden/orden.aggregate.ts
import { randomUUID } from "node:crypto";

// Value Objects
class Dinero {
  private constructor(
    public readonly monto: number,
    public readonly moneda: string,
  ) {
    if (monto <= 0) throw new Error("Monto debe ser positivo");
  }
  static crear(monto: number, moneda: string = "EUR"): Dinero {
    return new Dinero(monto, moneda);
  }
}

// Entidad interna al Aggregate (no se accede directamente)
class LineaOrden {
  private constructor(
    public readonly id: string,
    public readonly producto: string,
    public readonly cantidad: number,
    public readonly precio: Dinero,
  ) {}
  static crear(producto: string, cantidad: number, precioUnitario: Dinero): LineaOrden {
    return new LineaOrden(randomUUID(), producto, cantidad, precioUnitario);
  }
  get subtotal(): Dinero {
    return Dinero.crear(this.precio.monto * this.cantidad, this.precio.moneda);
  }
}

// Aggregate Root
export class Orden {
  private lineas: LineaOrden[] = [];
  private eventos: DomainEvent[] = [];

  private constructor(
    public readonly id: string,
    public readonly clienteId: string,
    private _estado: "borrador" | "confirmada" | "pagada" | "cancelada",
    private _fechaCreacion: Date,
  ) {}

  static crear(clienteId: string): Orden {
    const orden = new Orden(randomUUID(), clienteId, "borrador", new Date());
    orden.eventos.push(new OrdenCreadaEvent(orden.id, clienteId));
    return orden;
  }

  get estado() { return this._estado; }
  get fechaCreacion() { return this._fechaCreacion; }
  get lineas(): readonly LineaOrden[] { return this.lineas; }

  agregarLinea(producto: string, cantidad: number, precio: Dinero): void {
    if (this._estado !== "borrador") throw new Error("Solo se pueden agregar lineas a ordenes en borrador");
    this.lineas.push(LineaOrden.crear(producto, cantidad, precio));
  }

  get total(): Dinero {
    return Dinero.crear(
      this.lineas.reduce((sum, l) => sum + l.subtotal.monto, 0),
      "EUR",
    );
  }

  confirmar(): void {
    if (this.lineas.length === 0) throw new Error("Orden sin lineas");
    this._estado = "confirmada";
    this.eventos.push(new OrdenConfirmadaEvent(this.id, this.total.monto));
  }

  // Extraer eventos para publicar (sin exponer la lista mutable)
  pullEventos(): DomainEvent[] {
    const eventos = [...this.eventos];
    this.eventos = [];
    return eventos;
  }
}
```

---

## 14.9 Domain Events

```typescript
// src/domain/shared/domain-event.ts
export abstract class DomainEvent {
  public readonly ocurridoEn: Date = new Date();
  public readonly eventId: string = randomUUID();
  abstract readonly eventName: string;
}

export class OrdenCreadaEvent extends DomainEvent {
  readonly eventName = "orden.creada";
  constructor(public readonly ordenId: string, public readonly clienteId: string) {
    super();
  }
}

export class OrdenConfirmadaEvent extends DomainEvent {
  readonly eventName = "orden.confirmada";
  constructor(public readonly ordenId: string, public readonly total: number) {
    super();
  }
}

// Manejador de eventos (aplicacion)
export class EnviarFacturaCuandoOrdenConfirmada {
  async manejar(evento: OrdenConfirmadaEvent): Promise<void> {
    console.log(`Enviando factura para orden ${evento.ordenId} por ${evento.total} EUR`);
    // ... enviar email, generar PDF, etc.
  }
}

// Dispatcher (infraestructura)
export class EventDispatcher {
  private handlers = new Map<string, Array<(event: DomainEvent) => Promise<void>>>();

  registrar(eventName: string, handler: (event: DomainEvent) => Promise<void>): void {
    if (!this.handlers.has(eventName)) this.handlers.set(eventName, []);
    this.handlers.get(eventName)!.push(handler);
  }

  async despachar(eventos: DomainEvent[]): Promise<void> {
    for (const evento of eventos) {
      const handlers = this.handlers.get(evento.eventName) ?? [];
      await Promise.all(handlers.map((h) => h(evento)));
    }
  }
}
```

---

## 14.10 Bounded Contexts

```
┌─────────────────────┐    ┌─────────────────────┐
│   Contexto: Ventas   │    │  Contexto: Envios    │
│                     │    │                     │
│  Orden (Aggregate)  │───▶│  Envio (Aggregate)  │
│  Cliente (entidad)  │    │  Transportista       │
│  Producto (VO)      │    │  Direccion (VO)      │
└─────────────────────┘    └─────────────────────┘

Cada bounded context tiene su propio lenguaje ubicuo (ubiquitous language):
- Ventas: "Cliente", "Orden", "Producto", "Precio"
- Envios: "Destinatario", "Envio", "Bulto", "Ruta"

La misma entidad puede tener distintos nombres y propiedades en cada contexto.
```

---

## Resumen del Capítulo

- La arquitectura hexagonal aísla el dominio de la infraestructura mediante puertos (interfaces) y adaptadores (implementaciones).
- El dominio contiene Value Objects, Entidades y reglas de negocio. Cero dependencias externas.
- La capa de aplicacion orquesta casos de uso usando los puertos (interfaces).
- La infraestructura implementa adaptadores: Prisma, Express, Nodemailer, Redis.
- La composicion raiz (index.ts) ensambla todas las dependencias.
- El testing es trivial: dominio sin mocks, aplicacion con mocks de puertos.
- TypeScript brilla modelando el dominio con tipos precisos y value objects inmutables.

En el siguiente capítulo exploraremos temas avanzados de Node.js y TypeScript.

---

← [Capítulo anterior](13-patrones-de-diseno.md) | [Inicio](README.md) | [Capítulo siguiente →](15-temas-avanzados.md)
