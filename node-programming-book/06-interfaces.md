# Capítulo 6: Interfaces y Tipos Avanzados

TypeScript ofrece uno de los sistemas de tipos mas expresivos de cualquier lenguaje mainstream. Las interfaces y los type aliases son los pilares para modelar contratos, componer tipos y crear abstracciones reutilizables.

---

## 6.1 Interfaces

### Definicion e implementacion

```typescript
interface Figura {
  readonly id: string;
  area(): number;
  perimetro(): number;
}

// Implementacion implicita (structural typing)
const circulo: Figura = {
  id: "c1",
  area() {
    return Math.PI * 5 ** 2;
  },
  perimetro() {
    return 2 * Math.PI * 5;
  },
};

// Implementacion explicita con clases
class Rectangulo implements Figura {
  readonly id: string;

  constructor(
    id: string,
    private ancho: number,
    private alto: number,
  ) {
    this.id = id;
  }

  area(): number {
    return this.ancho * this.alto;
  }

  perimetro(): number {
    return 2 * (this.ancho + this.alto);
  }
}

// TypeScript usa duck typing estructural, no nominal
// Si un objeto tiene las mismas propiedades y metodos, implementa la interfaz
```

### Interfaces para funciones y objetos hibridos

```typescript
// Call signature: interfaz describe una funcion
interface Comparador<T> {
  (a: T, b: T): number;
}

const porNombre: Comparador<{ nombre: string }> = (a, b) =>
  a.nombre.localeCompare(b.nombre);

// Index signature: propiedades dinamicas
interface Cache {
  [clave: string]: unknown;
  ttl: number;  // Propiedad fija
  limpiar(): void;
}

const cache: Cache = {
  ttl: 3000,
  "user:1": { id: 1, nombre: "Andres" },
  "user:2": { id: 2, nombre: "Maria" },
  limpiar() {
    for (const clave of Object.keys(this)) {
      if (clave !== "ttl" && clave !== "limpiar") {
        delete this[clave as keyof this];
      }
    }
  },
};
```

### Extender interfaces (herencia)

```typescript
interface Entidad {
  id: number;
  creadoEn: Date;
}

interface ConNombre {
  nombre: string;
}

// Herencia multiple
interface Usuario extends Entidad, ConNombre {
  email: string;
  edad?: number;
}

// El tipo Usuario tiene: id, creadoEn, nombre, email, edad?

// Extender para especializar
interface Admin extends Usuario {
  nivel: number;
  permisos: string[];
}

const admin: Admin = {
  id: 1,
  creadoEn: new Date(),
  nombre: "Admin",
  email: "admin@test.com",
  nivel: 3,
  permisos: ["crear", "leer", "actualizar", "eliminar"],
};
```

---

## 6.2 Type Aliases vs Interfaces

### Cuando usar cada uno

```typescript
// INTERFACE: para describir formas de objetos
// Ventajas: se pueden extender (extends), hacer merge (declaration merging)
interface Usuario {
  id: number;
  nombre: string;
}

interface Usuario {
  email: string; // Merge: se agrega al Usuario existente
}

// TYPE ALIAS: para todo lo demas
// Unions, intersections, tuples, primitivas, funciones

type ID = string | number;
type Punto = [number, number];
type Operacion = (a: number, b: number) => number;
type Estado = "activo" | "inactivo" | "pendiente";

// Tambien para objetos (pero no se puede hacer merge)
type Producto = {
  id: number;
  nombre: string;
  precio: number;
};
```

### Tabla comparativa

| Caracteristica | Interface | Type Alias |
|---------------|-----------|------------|
| Describir objetos | ✅ | ✅ |
| Extender | `extends` | `&` (intersection) |
| Declaration merging | ✅ | ❌ |
| Union types | ❌ | ✅ |
| Tuplas | ❌ | ✅ |
| Primitivas | ❌ | ✅ |
| Mapped types | ❌ | ✅ |
| Rendimiento | Ligeramente mejor | Igual |

**Regla practica**: usa `interface` para APIs publicas de objetos (extensibilidad). Usa `type` para unions, tuplas, y cuando necesites composicion avanzada.

---

## 6.3 Intersection y Union Types Avanzados

```typescript
// Union types: A O B
type Resultado = { ok: true; datos: string } | { ok: false; error: string };

function manejarResultado(r: Resultado): string {
  if (r.ok) {
    return r.datos;      // Aqui r es { ok: true; datos: string }
  }
  return r.error;         // Aqui r es { ok: false; error: string }
}

// Intersection types: A Y B
interface ConTimestamp {
  creadoEn: Date;
  actualizadoEn: Date;
}

interface ConAutor {
  autorId: number;
}

type Post = { titulo: string; contenido: string } & ConTimestamp & ConAutor;

const post: Post = {
  titulo: "Mi Post",
  contenido: "Contenido...",
  creadoEn: new Date(),
  actualizadoEn: new Date(),
  autorId: 1,
};

// Union de tipos literales con discriminante
type Evento =
  | { tipo: "click"; x: number; y: number }
  | { tipo: "tecla"; tecla: string }
  | { tipo: "scroll"; deltaY: number };

function manejarEvento(evento: Evento): void {
  switch (evento.tipo) {
    case "click":
      console.log(`Click en (${evento.x}, ${evento.y})`);
      break;
    case "tecla":
      console.log(`Tecla presionada: ${evento.tecla}`);
      break;
    case "scroll":
      console.log(`Scroll: ${evento.deltaY}px`);
      break;
  }
}
```

---

## 6.4 Discriminated Unions (Uniones Discriminadas)

El patron mas poderoso de TypeScript para modelar estados:

```typescript
// Patron: estado de carga
type EstadoAsync<T> =
  | { estado: "idle" }
  | { estado: "loading" }
  | { estado: "success"; datos: T }
  | { estado: "error"; mensaje: string; codigo?: number };

// Hook o funcion que maneja el estado
function renderizarEstado<T>(estado: EstadoAsync<T>): string {
  switch (estado.estado) {
    case "idle":
      return "Sin iniciar";
    case "loading":
      return "Cargando...";
    case "success":
      return `Datos: ${JSON.stringify(estado.datos)}`;
    case "error":
      return `Error${estado.codigo ? ` ${estado.codigo}` : ""}: ${estado.mensaje}`;
    default:
      return exhaustivenessCheck(estado);
  }
}

// Funcion helper para exhaustiveness checking
function exhaustivenessCheck(valor: never): never {
  throw new Error(`Estado no manejado: ${JSON.stringify(valor)}`);
}

// Ejemplo practico: sistema de notificaciones
type Notificacion =
  | { tipo: "exito"; titulo: string }
  | { tipo: "error"; titulo: string; descripcion: string }
  | { tipo: "info"; mensaje: string; duracion?: number }
  | { tipo: "warning"; mensaje: string; accion?: () => void };

function mostrarNotificacion(notif: Notificacion): void {
  const estilos = {
    exito: "🟢",
    error: "🔴",
    info: "🔵",
    warning: "🟡",
  };

  console.log(`${estilos[notif.tipo]} ${obtenerMensaje(notif)}`);
}

function obtenerMensaje(notif: Notificacion): string {
  switch (notif.tipo) {
    case "exito": return notif.titulo;
    case "error": return `${notif.titulo}: ${notif.descripcion}`;
    case "info": return notif.mensaje;
    case "warning": return notif.mensaje;
  }
}
```

---

## 6.5 Conditional Types

Tipos que dependen de condiciones:

```typescript
// Conditional type: T extends U ? X : Y
type EsString<T> = T extends string ? true : false;

type A = EsString<"hola">;   // true
type B = EsString<number>;   // false

// Extraer tipos de una union
type ExtraerString<T> = T extends string ? T : never;
type SoloStrings = ExtraerString<string | number | boolean>; // string

// Excluir tipos de una union (built-in)
type SoloNumeros = Exclude<string | number | boolean, string | boolean>; // number

// Extraer de una union (built-in)
type SoloFunciones = Extract<string | (() => void) | number, (...args: any[]) => any>;

// Inferencia con conditional types
type Retorno<T> = T extends (...args: any[]) => infer R ? R : never;
type ResultadoSuma = Retorno<(a: number, b: number) => string>; // string

// Desempaquetar array
type ElementoArray<T> = T extends (infer E)[] ? E : never;
type TipoElemento = ElementoArray<string[]>; // string

// Desempaquetar Promise
type Awaited<T> = T extends Promise<infer U> ? U : T;
type ResultadoPromesa = Awaited<Promise<number>>; // number
// Nota: TypeScript ya incluye Awaited<T> como utility type
```

---

## 6.6 Mapped Types

Transforman propiedades de un tipo existente:

```typescript
interface Usuario {
  id: number;
  nombre: string;
  email: string;
  edad: number;
}

// Todas las propiedades opcionales
type UsuarioParcial = {
  [K in keyof Usuario]?: Usuario[K];
};

// Todas las propiedades readonly
type UsuarioReadonly = {
  readonly [K in keyof Usuario]: Usuario[K];
};

// Transformar tipos de propiedades
type UsuarioString = {
  [K in keyof Usuario]: string;
};
// { id: string; nombre: string; email: string; edad: string }

// Filtrar propiedades por tipo
type SoloStrings<T> = {
  [K in keyof T as T[K] extends string ? K : never]: T[K];
};

type UsuarioStrings = SoloStrings<Usuario>;
// { nombre: string; email: string } - id y edad se excluyen

// Agregar prefijo a claves
type ConPrefijo<T, P extends string> = {
  [K in keyof T as `${P}${Capitalize<string & K>}`]: T[K];
};

type UsuarioPrefijado = ConPrefijo<Usuario, "usuario">;
// { usuarioId: number; usuarioNombre: string; ... }
```

---

## 6.7 Template Literal Types

```typescript
// Combinar literales para crear nuevos tipos
type MetodoHTTP = "GET" | "POST" | "PUT" | "DELETE";
type Recurso = "usuarios" | "productos" | "ordenes";

type RutaAPI = `/${Recurso}`;                    // "/usuarios" | "/productos" | "/ordenes"
type RutaConID = `/${Recurso}/${number}`;         // "/usuarios/123" | ...
type Endpoint = `${MetodoHTTP} /${Recurso}`;      // "GET /usuarios" | ...

// Funcion con ruta tipada
function crearRuta(recurso: Recurso, id?: number): string {
  if (id !== undefined) {
    return `/${recurso}/${id}` as const;
  }
  return `/${recurso}` as const;
}

// Utilidades de string en tipos
type EventoConPrefijo = `on${Capitalize<string>}`;
// `on${Capitalize<"click">}` = "onClick"

type CSSPropiedad = "margin-top" | "padding-left" | "border-right";
type CSSPropiedadCamel = CamelCase<CSSPropiedad>;
// Necesitas implementar CamelCase con template literal types

// Implementacion de CamelCase
type CamelCase<S extends string> = S extends `${infer P}-${infer R}`
  ? `${P}${Capitalize<CamelCase<R>>}`
  : S;
```

---

## 6.8 Branded Types (Tipos Nominales)

TypeScript usa tipado estructural, pero puedes simular tipado nominal:

```typescript
// Branded type: tipo estructural con una marca unica
type UsuarioID = string & { readonly __brand: "UsuarioID" };
type OrdenID = string & { readonly __brand: "OrdenID" };

function crearUsuarioID(id: string): UsuarioID {
  return id as UsuarioID;
}

function crearOrdenID(id: string): OrdenID {
  return id as OrdenID;
}

function buscarUsuario(id: UsuarioID): void {
  console.log(`Buscando usuario ${id}`);
}

function buscarOrden(id: OrdenID): void {
  console.log(`Buscando orden ${id}`);
}

const usuarioID = crearUsuarioID("abc-123");
const ordenID = crearOrdenID("abc-123");

buscarUsuario(usuarioID);  // OK
// buscarUsuario(ordenID); // ❌ Error: OrdenID no es UsuarioID
// buscarUsuario("abc-123"); // ❌ Error: string no es UsuarioID

// Branded types con generics
type Brand<T, B> = T & { __brand: B };

type Email = Brand<string, "Email">;
type Password = Brand<string, "Password">;

function crearEmail(email: string): Email {
  if (!email.includes("@")) throw new Error("Email invalido");
  return email as Email;
}

function login(email: Email, password: Password): void {
  console.log(`Login: ${email}`);
}
```

---

## 6.9 Module Augmentation (Extender tipos externos)

```typescript
// Extender tipos de librerias sin modificar sus fuentes

// Ejemplo 1: agregar propiedades a Express Request
// En un archivo .d.ts de tu proyecto:
declare global {
  namespace Express {
    interface Request {
      usuarioId: string;
      sesion: { token: string; expira: Date };
    }
  }
}

// Ahora en cualquier handler:
app.get("/perfil", (req: Request, res: Response) => {
  const id = req.usuarioId; // TypeScript lo reconoce
});

// Ejemplo 2: extender interfaces de librerias
import "fastify";

declare module "fastify" {
  interface FastifyRequest {
    usuarioId: string;
  }
  interface FastifyInstance {
    db: PrismaClient; // Decorador personalizado
  }
}

// Ejemplo 3: agregar tipos a modulos sin tipos
declare module "mi-libreria-sin-tipos" {
  export function saludar(nombre: string): string;
  export const VERSION: string;
}
```

---

## 6.10 Recursive Types y Tipos Anidados

```typescript
// Tipos recursivos: el tipo se referencia a si mismo
type JSONValue =
  | string
  | number
  | boolean
  | null
  | JSONValue[]
  | { [key: string]: JSONValue };

const datos: JSONValue = {
  nombre: "Andres",
  edad: 30,
  activo: true,
  direcciones: [
    { calle: "Principal", numero: 123 },
    { calle: "Secundaria", numero: 456 },
  ],
  metadata: null,
};

// Arbol generico
interface Nodo<T> {
  valor: T;
  izquierda: Nodo<T> | null;
  derecha: Nodo<T> | null;
}

const arbol: Nodo<number> = {
  valor: 5,
  izquierda: { valor: 3, izquierda: null, derecha: null },
  derecha: { valor: 7, izquierda: null, derecha: null },
};

// Tipo para rutas de un objeto (deep paths)
type Path<T, P extends string = ""> = T extends object
  ? {
      [K in keyof T]: K extends string
        ? Path<T[K], `${P}${P extends "" ? "" : "."}${K}`>
        : never;
    }[keyof T]
  : P;

// type RutasUsuario = "nombre" | "email" | "direccion.calle" | "direccion.ciudad"
```

---

## 6.11 Variance Annotations (TypeScript 4.7+)

```typescript
// Variance: controla subtipado en generics

// Invariante (por defecto): ni covariant ni contravariant
interface Repository<T> {
  get(id: string): T;           // T en posicion de salida (covariant)
  save(item: T): void;          // T en posicion de entrada (contravariant)
}
// Repository<Admin> NO es subtipo de Repository<Usuario>
// aunque Admin extiende Usuario

// Covariant (out): solo posiciones de salida
interface ReadOnlyRepository<out T> {
  get(id: string): T;
  find(predicate: (item: T) => boolean): T | undefined;
  // save(item: T): void;  // ❌ Error: T no puede estar en entrada
}
// ReadOnlyRepository<Admin> SI es subtipo de ReadOnlyRepository<Usuario>

// Contravariant (in): solo posiciones de entrada
interface WriteOnlyRepository<in T> {
  save(item: T): void;
  delete(item: T): void;
  // get(id: string): T;  // ❌ Error: T no puede estar en salida
}
// WriteOnlyRepository<Usuario> SI es subtipo de WriteOnlyRepository<Admin>
```

### const type parameters (TypeScript 5.0+)

```typescript
// Sin const: infiere tipos amplios
function identidad<T>(valor: T): T {
  return valor;
}
const x = identidad({ nombre: "Andres" });
// x: { nombre: string }

// Con const: infiere tipos literales (como as const)
function identidadConst<const T>(valor: T): T {
  return valor;
}
const y = identidadConst({ nombre: "Andres" });
// y: { readonly nombre: "Andres" }

// Muy util con tuplas
function tupla<const T extends readonly unknown[]>(...args: T): T {
  return args;
}
const t = tupla("hola", 42, true);
// t: readonly ["hola", 42, true]  (tipos literales exactos)
```

---

## 6.12 Explicit Resource Management (using)

```typescript
// TypeScript 5.2+ / ES2024: liberar recursos automaticamente

// Symbol.dispose (sincrono)
class ConexionBD {
  constructor(private dsn: string) {
    console.log(`Conectando a ${dsn}...`);
  }

  [Symbol.dispose](): void {
    console.log("Cerrando conexion (dispose)");
  }

  query(sql: string) {
    console.log(`Ejecutando: ${sql}`);
  }
}

function usarBD() {
  using conn = new ConexionBD("postgres://localhost/mi_db");
  conn.query("SELECT * FROM usuarios");
  // conn se libera automaticamente al salir del scope
}

// Symbol.asyncDispose (asincrono)
class ConexionRedis {
  private client: any;

  async [Symbol.asyncDispose](): Promise<void> {
    console.log("Cerrando Redis (async dispose)");
    await this.client.quit();
  }

  async get(key: string) {
    return `valor_de_${key}`;
  }
}

async function usarRedis() {
  await using redis = new ConexionRedis();
  const valor = await redis.get("clave");
  console.log(valor);
  // redis se libera automaticamente (incluso si hay error)
}

// Requiere: "target": "ES2022" y lib: ["ESNext", "ESNext.Disposable"]
```

---

## Resumen del Capítulo

- Interfaces definen contratos para objetos. Implementacion estructural (duck typing).
- Type aliases son mas flexibles: unions, intersections, tuplas, primitivas.
- Regla: `interface` para APIs publicas, `type` para composicion y unions.
- Discriminated unions modelan estados finitos de forma exhaustiva y segura.
- Conditional types (`T extends U ? X : Y`) crean tipos que dependen de condiciones.
- Mapped types transforman propiedades existentes (`{ [K in keyof T]: ... }`).
- Template literal types combinan string literales para crear tipos de string complejos.
- Branded types simulan tipado nominal para prevenir mezclar tipos estructuralmente identicos.
- Utility types (`Partial`, `Pick`, `Omit`, `Record`) cubren la mayoria de transformaciones comunes.

En el siguiente capitulo exploraremos la asincronia y el Event Loop en Node.js.

---

← [Capítulo anterior](05-estructuras-de-datos.md) | [Inicio](README.md) | [Capítulo siguiente →](07-asincronia.md)
