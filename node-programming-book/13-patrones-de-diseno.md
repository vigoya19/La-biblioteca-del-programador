# Capítulo 13: Patrones de Diseño en Node.js

Los patrones de diseno son soluciones probadas a problemas recurrentes. En Node.js y TypeScript, los patrones se adaptan al ecosistema asincrono, la flexibilidad de JavaScript y la seguridad de tipos.

---

## 13.1 Patrones Creacionales

### Factory (Fabrica)

```typescript
interface Notificador {
  enviar(mensaje: string): Promise<void>;
}

class EmailNotifier implements Notificador {
  async enviar(mensaje: string): Promise<void> {
    console.log(`Enviando email: ${mensaje}`);
  }
}

class SMSNotifier implements Notificador {
  async enviar(mensaje: string): Promise<void> {
    console.log(`Enviando SMS: ${mensaje}`);
  }
}

class PushNotifier implements Notificador {
  async enviar(mensaje: string): Promise<void> {
    console.log(`Enviando push: ${mensaje}`);
  }
}

// Factory function
type TipoNotificacion = "email" | "sms" | "push";

function crearNotificador(tipo: TipoNotificacion): Notificador {
  switch (tipo) {
    case "email": return new EmailNotifier();
    case "sms":   return new SMSNotifier();
    case "push":  return new PushNotifier();
    default: throw new Error(`Tipo desconocido: ${tipo}`);
  }
}
```

### Builder

```typescript
interface PeticionHTTP {
  metodo: "GET" | "POST" | "PUT" | "DELETE";
  url: string;
  headers: Record<string, string>;
  cuerpo?: unknown;
  timeout: number;
}

class PeticionBuilder {
  private readonly peticion: Partial<PeticionHTTP> = {
    headers: {},
    timeout: 30_000,
  };

  constructor(private metodo: PeticionHTTP["metodo"], private url: string) {}

  header(clave: string, valor: string): this {
    this.peticion.headers![clave] = valor;
    return this;
  }

  bearerToken(token: string): this {
    return this.header("Authorization", `Bearer ${token}`);
  }

  json(cuerpo: unknown): this {
    this.peticion.cuerpo = cuerpo;
    return this.header("Content-Type", "application/json");
  }

  timeout(ms: number): this {
    this.peticion.timeout = ms;
    return this;
  }

  build(): PeticionHTTP {
    return {
      metodo: this.metodo,
      url: this.url,
      headers: this.peticion.headers!,
      cuerpo: this.peticion.cuerpo,
      timeout: this.peticion.timeout!,
    };
  }
}

// Uso
const peticion = new PeticionBuilder("POST", "/api/usuarios")
  .bearerToken("token-123")
  .json({ nombre: "Andres" })
  .timeout(5000)
  .build();
```

### Singleton

```typescript
class DatabaseConnection {
  private static instance: DatabaseConnection;
  private constructor(private connectionString: string) {}

  static getInstance(connectionString?: string): DatabaseConnection {
    if (!DatabaseConnection.instance) {
      if (!connectionString) {
        throw new Error("Connection string required for first init");
      }
      DatabaseConnection.instance = new DatabaseConnection(connectionString);
    }
    return DatabaseConnection.instance;
  }
}
```

---

## 13.2 Patrones Estructurales

### Adapter

```typescript
// API de terceros (Stripe)
class StripeAPI {
  createCharge(amount: number, currency: string) {
    console.log(`Stripe: charging ${amount} ${currency}`);
  }
}

// API de terceros (PayPal)
class PayPalSDK {
  makePayment(amount: number, currencyCode: string) {
    console.log(`PayPal: payment of ${amount} ${currencyCode}`);
  }
}

// Interface comun
interface PaymentProcessor {
  pay(amount: number, currency: string): void;
}

// Adapters
class StripeAdapter implements PaymentProcessor {
  constructor(private stripe: StripeAPI) {}
  pay(amount: number, currency: string): void {
    const amountInCents = Math.round(amount * 100);
    this.stripe.createCharge(amountInCents, currency);
  }
}

class PayPalAdapter implements PaymentProcessor {
  constructor(private paypal: PayPalSDK) {}
  pay(amount: number, currency: string): void {
    this.paypal.makePayment(amount, currency);
  }
}

// Uso: mismo codigo, diferente backend
function procesarPago(processor: PaymentProcessor) {
  processor.pay(99.99, "USD");
}

procesarPago(new StripeAdapter(new StripeAPI()));
procesarPago(new PayPalAdapter(new PayPalSDK()));
```

### Decorator (middleware pattern)

```typescript
import { Request, Response, NextFunction } from "express";

type Middleware = (req: Request, res: Response, next: NextFunction) => void;

function withLogging(handler: Middleware): Middleware {
  return (req, res, next) => {
    const inicio = Date.now();
    console.log(`${req.method} ${req.path}`);
    handler(req, res, () => {
      console.log(`${req.path} ${Date.now() - inicio}ms`);
      next();
    });
  };
}

function withAuth(handler: Middleware): Middleware {
  return (req, res, next) => {
    if (!req.headers.authorization) {
      return res.status(401).json({ error: "No autorizado" });
    }
    handler(req, res, next);
  };
}

// Composicion
function composeMiddleware(...middlewares: Middleware[]): Middleware {
  return middlewares.reduceRight(
    (composed, m) => (req, res, next) => m(req, res, () => composed(req, res, next)),
    (req, res, next) => next(),
  );
}
```

---

## 13.3 Patrones de Comportamiento

### Strategy

```typescript
interface EstrategiaCache {
  get(key: string): Promise<string | null>;
  set(key: string, value: string, ttl?: number): Promise<void>;
  delete(key: string): Promise<void>;
}

class RedisStrategy implements EstrategiaCache {
  constructor(private redis: Redis) {}
  async get(key: string) { return this.redis.get(key); }
  async set(key: string, value: string, ttl?: number) {
    if (ttl) return this.redis.setex(key, ttl, value);
    return this.redis.set(key, value);
  }
  async delete(key: string) { await this.redis.del(key); }
}

class MemoryStrategy implements EstrategiaCache {
  private store = new Map<string, { value: string; expiresAt?: number }>();
  async get(key: string) { /* ... */ }
  async set(key: string, value: string, ttl?: number) { /* ... */ }
  async delete(key: string) { this.store.delete(key); }
}

class CacheService {
  constructor(private strategy: EstrategiaCache) {}
  async get(key: string) { return this.strategy.get(key); }
  async set(key: string, value: string, ttl?: number) { return this.strategy.set(key, value, ttl); }
}
```

### Observer (EventEmitter nativo)

```typescript
import { EventEmitter } from "node:events";

interface EventosApp {
  "usuario:creado": (usuario: { id: string; email: string }) => void;
  "usuario:eliminado": (id: string) => void;
  "orden:completada": (ordenId: string, total: number) => void;
}

class EventBus extends EventEmitter {
  on<E extends keyof EventosApp>(event: E, listener: EventosApp[E]): this {
    return super.on(event, listener);
  }

  emit<E extends keyof EventosApp>(event: E, ...args: Parameters<EventosApp[E]>): boolean {
    return super.emit(event, ...args);
  }
}

const bus = new EventBus();

// Suscriptor: enviar email de bienvenida
bus.on("usuario:creado", (usuario) => {
  console.log(`Email de bienvenida a ${usuario.email}`);
});

// Suscriptor: crear metricas
bus.on("usuario:creado", (usuario) => {
  console.log(`Metrica: usuario_creado id=${usuario.id}`);
});

// Emisor
bus.emit("usuario:creado", { id: "123", email: "user@test.com" });
```

---

## 13.4 Patrones Especificos de Node.js

### Middleware Pattern

```typescript
type Handler = (contexto: Contexto) => Promise<void>;
type MiddlewareFn = (contexto: Contexto, next: () => Promise<void>) => Promise<void>;

class Pipeline {
  private middlewares: MiddlewareFn[] = [];

  use(mw: MiddlewareFn): this {
    this.middlewares.push(mw);
    return this;
  }

  async ejecutar(contexto: Contexto, handler: Handler): Promise<void> {
    let index = 0;

    const next = async (): Promise<void> => {
      if (index < this.middlewares.length) {
        const middleware = this.middlewares[index++];
        await middleware(contexto, next);
      } else {
        await handler(contexto);
      }
    };

    await next();
  }
}
```

### Circuit Breaker

```typescript
class CircuitBreaker {
  private estado: "cerrado" | "abierto" | "semi-abierto" = "cerrado";
  private fallos = 0;
  private ultimoFallo = 0;

  constructor(
    private maxFallos: number,
    private timeout: number,
  ) {}

  async ejecutar<T>(fn: () => Promise<T>): Promise<T> {
    if (this.estado === "abierto") {
      if (Date.now() - this.ultimoFallo < this.timeout) {
        throw new Error("Circuito abierto");
      }
      this.estado = "semi-abierto";
    }

    try {
      const resultado = await fn();
      this.exito();
      return resultado;
    } catch (error) {
      this.fallo();
      throw error;
    }
  }

  private exito(): void {
    this.fallos = 0;
    this.estado = "cerrado";
  }

  private fallo(): void {
    this.fallos++;
    this.ultimoFallo = Date.now();
    if (this.fallos >= this.maxFallos) {
      this.estado = "abierto";
    }
  }
}
```

### Retry Pattern

```typescript
async function retry<T>(
  fn: () => Promise<T>,
  opciones: { intentos: number; esperaMs: number; backoff?: "exponencial" },
): Promise<T> {
  for (let i = 0; i < opciones.intentos; i++) {
    try {
      return await fn();
    } catch (error) {
      if (i === opciones.intentos - 1) throw error;
      const espera = opciones.backoff === "exponencial"
        ? opciones.esperaMs * Math.pow(2, i)
        : opciones.esperaMs;
      console.log(`Intento ${i + 1} fallido, reintentando en ${espera}ms`);
      await new Promise((r) => setTimeout(r, espera));
    }
  }
  throw new Error("Inalcanzable");
}
```

---

## 13.5 Proxy Pattern

```typescript
// Proxy: controla acceso a otro objeto
interface ServicioPesado {
  obtenerDatos(id: string): Promise<{ id: string; valor: string }>;
}

class ServicioPesadoReal implements ServicioPesado {
  async obtenerDatos(id: string) {
    console.log(`Consulta costosa a BD para ${id}...`);
    // Simular delay de red/BD
    await new Promise((r) => setTimeout(r, 1000));
    return { id, valor: `datos_de_${id}` };
  }
}

// Proxy con cache
class ServicioCacheProxy implements ServicioPesado {
  private cache = new Map<string, { id: string; valor: string }>();

  constructor(private real: ServicioPesadoReal) {}

  async obtenerDatos(id: string) {
    if (this.cache.has(id)) {
      console.log(`Cache HIT para ${id}`);
      return this.cache.get(id)!;
    }
    console.log(`Cache MISS para ${id}`);
    const datos = await this.real.obtenerDatos(id);
    this.cache.set(id, datos);
    return datos;
  }
}

// Proxy con rate limiting
class ServicioRateLimitProxy implements ServicioPesado {
  private peticiones = 0;
  private ventanaInicio = Date.now();

  constructor(
    private real: ServicioPesadoReal,
    private maxPeticiones: number,
  ) {}

  async obtenerDatos(id: string) {
    if (Date.now() - this.ventanaInicio > 60_000) {
      this.peticiones = 0;
      this.ventanaInicio = Date.now();
    }
    if (++this.peticiones > this.maxPeticiones) {
      throw new Error("Rate limit excedido");
    }
    return this.real.obtenerDatos(id);
  }
}
```

---

## 13.6 Facade Pattern

```typescript
// Facade: simplifica un subsistema complejo
class ServicioAutenticacion {
  async login(email: string, password: string): Promise<string> { return "token"; }
}

class ServicioPagos {
  async cobrar(token: string, monto: number): Promise<string> { return "tx-123"; }
}

class ServicioNotificaciones {
  async enviarConfirmacion(email: string, transaccion: string): Promise<void> {}
}

// Facade: expone una API simple para el cliente
class TiendaFacade {
  constructor(
    private auth: ServicioAutenticacion,
    private pagos: ServicioPagos,
    private notif: ServicioNotificaciones,
  ) {}

  async comprar(email: string, password: string, monto: number) {
    const token = await this.auth.login(email, password);
    const tx = await this.pagos.cobrar(token, monto);
    await this.notif.enviarConfirmacion(email, tx);
    return { transaccion: tx, monto };
  }
}

// Cliente: una sola llamada, no 3
const tienda = new TiendaFacade(new ServicioAutenticacion(), new ServicioPagos(), new ServicioNotificaciones());
const resultado = await tienda.comprar("user@test.com", "pass", 99.99);
```

---

## 13.7 Command Pattern

```typescript
interface Comando {
  ejecutar(): Promise<void>;
  deshacer(): Promise<void>;
}

class CrearUsuarioComando implements Comando {
  private usuarioCreado?: { id: string; email: string };

  constructor(private email: string, private nombre: string, private repo: UsuarioRepository) {}

  async ejecutar() {
    this.usuarioCreado = await this.repo.crear({ email: this.email, nombre: this.nombre });
  }

  async deshacer() {
    if (this.usuarioCreado) {
      await this.repo.eliminar(this.usuarioCreado.id);
    }
  }
}

// Invocador con historial (undo)
class HistorialComandos {
  private ejecutados: Comando[] = [];

  async ejecutar(comando: Comando) {
    await comando.ejecutar();
    this.ejecutados.push(comando);
  }

  async deshacerUltimo() {
    const comando = this.ejecutados.pop();
    if (comando) await comando.deshacer();
  }
}
```

---

## 13.8 Chain of Responsibility

```typescript
type HandlerFn = (peticion: Peticion) => Peticion | null;
type MiddlewareFn = (peticion: Peticion, next: () => Promise<Peticion>) => Promise<Peticion>;

interface Peticion {
  body: unknown;
  headers: Record<string, string>;
  usuarioId?: string;
}

class ValidationHandler {
  ejecutar(peticion: Peticion, next: () => Promise<Peticion>): Promise<Peticion> {
    if (!peticion.body) throw new Error("Body requerido");
    return next();
  }
}

class AuthHandler {
  ejecutar(peticion: Peticion, next: () => Promise<Peticion>): Promise<Peticion> {
    const token = peticion.headers["authorization"];
    if (!token) throw new Error("No autorizado");
    peticion.usuarioId = verificarToken(token);
    return next();
  }
}

class BusinessHandler {
  ejecutar(peticion: Peticion, next: () => Promise<Peticion>): Promise<Peticion> {
    console.log("Procesando logica de negocio para", peticion.usuarioId);
    return next();
  }
}

// Encadenar handlers
class Pipeline {
  private handlers: Array<{ ejecutar: HandlerFn }> = [];

  use(h: { ejecutar: HandlerFn }): this { this.handlers.push(h); return this; }

  async process(peticion: Peticion): Promise<Peticion> {
    let index = 0;
    const next = async (): Promise<Peticion> => {
      if (index >= this.handlers.length) return peticion;
      const handler = this.handlers[index++];
      return handler.ejecutar(peticion, next);
    };
    return next();
  }
}
```

---

## 13.9 Pub/Sub Pattern

```typescript
type Suscriptor<T> = (datos: T) => void;

class EventBus<Topics extends Record<string, unknown>> {
  private suscriptores = new Map<keyof Topics, Set<Suscriptor<any>>>();

  on<T extends keyof Topics>(topico: T, fn: Suscriptor<Topics[T]>): void {
    if (!this.suscriptores.has(topico)) this.suscriptores.set(topico, new Set());
    this.suscriptores.get(topico)!.add(fn);
  }

  off<T extends keyof Topics>(topico: T, fn: Suscriptor<Topics[T]>): void {
    this.suscriptores.get(topico)?.delete(fn);
  }

  emit<T extends keyof Topics>(topico: T, datos: Topics[T]): void {
    for (const fn of this.suscriptores.get(topico) ?? []) {
      fn(datos);
    }
  }
}

// Uso
type Eventos = {
  "usuario:creado": { id: string; email: string };
  "orden:completada": { ordenId: string; total: number };
};

const bus = new EventBus<Eventos>();
bus.on("usuario:creado", (u) => console.log(`Bienvenida a ${u.email}`));
bus.on("orden:completada", (o) => console.log(`Facturar ${o.total}`));
bus.emit("usuario:creado", { id: "1", email: "test@test.com" });
```

---

## Resumen del Capítulo

- **Factory**: crea objetos sin exponer logica de creacion. Funciones factory en vez de clases abstractas.
- **Builder**: construye objetos paso a paso con API fluida. Ideal para peticiones HTTP, queries.
- **Singleton**: una instancia unica. En Node.js los modulos son naturalmente singleton.
- **Adapter**: traduce interfaces para integrar APIs de terceros o sistemas legacy.
- **Decorator/Middleware**: envuelve handlers para agregar comportamiento (auth, logging, timing).
- **Strategy**: intercambia algoritmos en runtime via interfaces. Ideal para cache, pagos, storage.
- **Observer**: EventEmitter nativo de Node.js con tipos para eventos.
- **Circuit Breaker**: previene llamadas a servicios fallidos. Esencial en microservicios.
- **Retry**: reintenta operaciones fallidas con backoff exponencial.

En el siguiente capítulo exploraremos la arquitectura hexagonal y DDD con TypeScript.

---

← [Capítulo anterior](12-buenas-practicas.md) | [Inicio](README.md) | [Capítulo siguiente →](14-arquitectura-hexagonal.md)
