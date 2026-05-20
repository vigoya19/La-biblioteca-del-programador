# Capítulo 11: Generics en TypeScript

Los generics permiten escribir codigo reutilizable que funciona con multiples tipos manteniendo la seguridad de tipos. Son una de las herramientas mas poderosas de TypeScript para crear APIs flexibles y type-safe.

---

## 11.1 Funciones Genericas

```typescript
// Funcion identidad generica
function identidad<T>(valor: T): T {
  return valor;
}

// Inferencia automatica
const num = identidad(42);       // number
const str = identidad("hola");   // string

// Tipo explicito
const explicit = identidad<number>(42);

// Multiples parametros de tipo
function par<T, U>(primero: T, segundo: U): [T, U] {
  return [primero, segundo];
}

const miPar = par("hello", 42);  // [string, number]
```

### Constraints (Restricciones)

```typescript
// Restringir T a tipos que tengan ciertas propiedades
interface ConLongitud {
  length: number;
}

function logLongitud<T extends ConLongitud>(item: T): T {
  console.log(`Longitud: ${item.length}`);
  return item;
}

logLongitud("hola");          // OK: string tiene length
logLongitud([1, 2, 3]);       // OK: array tiene length
logLongitud({ length: 10 });  // OK: objeto con length
// logLongitud(42);            // ❌ Error: number no tiene length

// Constraint con keyof
function obtenerPropiedad<T, K extends keyof T>(obj: T, clave: K): T[K] {
  return obj[clave];
}

const usuario = { id: 1, nombre: "Andres", email: "a@test.com" };
const nombre = obtenerPropiedad(usuario, "nombre");  // string
// obtenerPropiedad(usuario, "apellido");  // ❌ Error
```

---

## 11.2 Generics en Clases e Interfaces

```typescript
// Clase generica
class Repositorio<T> {
  private items: Map<string, T> = new Map();

  guardar(id: string, item: T): void {
    this.items.set(id, item);
  }

  obtener(id: string): T | undefined {
    return this.items.get(id);
  }

  listar(): T[] {
    return [...this.items.values()];
  }

  eliminar(id: string): boolean {
    return this.items.delete(id);
  }
}

const repoUsuarios = new Repositorio<{ nombre: string }>();
repoUsuarios.guardar("1", { nombre: "Andres" });
const usuario = repoUsuarios.obtener("1"); // { nombre: string } | undefined

// Interfaz generica
interface Resultado<T, E = Error> {
  ok: boolean;
  value?: T;
  error?: E;
}

function exito<T>(value: T): Resultado<T> {
  return { ok: true, value };
}

function fallo<E = Error>(error: E): Resultado<never, E> {
  return { ok: false, error };
}

const r1 = exito(42);        // Resultado<number>
const r2 = fallo("error");   // Resultado<never, string>
```

### Stack y Queue genericas

```typescript
class Pila<T> {
  private items: T[] = [];

  push(item: T): void {
    this.items.push(item);
  }

  pop(): T | undefined {
    return this.items.pop();
  }

  get cima(): T | undefined {
    return this.items.at(-1);
  }

  get size(): number {
    return this.items.length;
  }

  *[Symbol.iterator](): Iterator<T> {
    for (const item of this.items) yield item;
  }
}
```

---

## 11.3 Utility Types Genericos

TypeScript incluye utility types que son generics aplicados:

```typescript
interface Usuario {
  id: number;
  nombre: string;
  email: string;
  edad: number;
  direccion?: string;
}

// Partial<T>: todas las propiedades opcionales
type ActualizacionUsuario = Partial<Usuario>;
// { id?: number; nombre?: string; ... }

// Required<T>: todas las propiedades obligatorias
type UsuarioCompleto = Required<Usuario>;
// direccion ahora es obligatoria

// Readonly<T>: propiedades de solo lectura
type UsuarioInmutable = Readonly<Usuario>;

// Pick<T, K>: seleccionar propiedades
type UsuarioBasico = Pick<Usuario, "id" | "nombre">;

// Omit<T, K>: excluir propiedades
type UsuarioSinID = Omit<Usuario, "id" | "edad">;

// Record<K, V>: diccionario tipado
type CacheUsuarios = Record<string, Usuario>;

// Exclude<T, U>: excluir de union
type SoloStrings = Exclude<string | number | boolean, number | boolean>;

// Extract<T, U>: extraer de union
type SoloFunciones = Extract<string | (() => void) | number, Function>;

// NonNullable<T>: excluir null y undefined
type ValoresValidos = NonNullable<string | null | undefined>; // string

// ReturnType<T>: tipo de retorno de funcion
function crearUsuario(nombre: string): Usuario {
  return { id: 1, nombre, email: `${nombre}@test.com`, edad: 0 };
}
type TipoUsuario = ReturnType<typeof crearUsuario>; // Usuario

// Parameters<T>: tipo de parametros de funcion
type ParametrosUsuario = Parameters<typeof crearUsuario>; // [nombre: string]

// Awaited<T>: desenvolver Promise
type ResultadoAsync = Awaited<Promise<Usuario>>; // Usuario
```

---

## 11.4 Patrones Avanzados con Generics

### Builder type-safe

```typescript
class Builder<T extends Record<string, unknown>> {
  private obj: Partial<T> = {};

  set<K extends keyof T>(key: K, value: T[K]): this {
    this.obj[key] = value;
    return this;
  }

  build(): T {
    return this.obj as T;
  }
}

interface ConfigServidor {
  puerto: number;
  host: string;
  tls: boolean;
  timeout: number;
}

const config = new Builder<ConfigServidor>()
  .set("puerto", 8080)
  .set("host", "localhost")
  .set("tls", true)
  .set("timeout", 5000)
  .build();
```

### Repository pattern generico

```typescript
interface Repository<T, ID = string> {
  findById(id: ID): Promise<T | null>;
  findAll(): Promise<T[]>;
  create(data: Omit<T, "id">): Promise<T>;
  update(id: ID, data: Partial<T>): Promise<T>;
  delete(id: ID): Promise<void>;
}

// Implementacion en memoria para tests
class InMemoryRepository<T extends { id: string }> implements Repository<T> {
  private items: Map<string, T> = new Map();

  async findById(id: string): Promise<T | null> {
    return this.items.get(id) ?? null;
  }

  async findAll(): Promise<T[]> {
    return [...this.items.values()];
  }

  async create(data: Omit<T, "id">): Promise<T> {
    const item = { ...data, id: crypto.randomUUID() } as unknown as T;
    this.items.set(item.id, item);
    return item;
  }

  async update(id: string, data: Partial<T>): Promise<T> {
    const existing = this.items.get(id);
    if (!existing) throw new Error(`Item ${id} not found`);
    const updated = { ...existing, ...data };
    this.items.set(id, updated);
    return updated;
  }

  async delete(id: string): Promise<void> {
    this.items.delete(id);
  }
}
```

---

## 11.5 Type Inference en Generics

```typescript
// Inferencia automatica
function primero<T>(arr: T[]): T | undefined {
  return arr[0];
}

const a = primero([1, 2, 3]);       // number | undefined
const b = primero(["a", "b"]);       // string | undefined

// La inferencia funciona con callbacks
function mapear<T, U>(arr: T[], fn: (item: T, index: number) => U): U[] {
  return arr.map(fn);
}

const dobles = mapear([1, 2, 3], (n) => n * 2);     // number[]
const textos = mapear([1, 2, 3], (n) => String(n));  // string[]

// Cuando TypeScript NO puede inferir
function crearArray<T>(longitud: number, valor: T): T[] {
  return Array.from({ length: longitud }, () => valor);
}

const arr = crearArray(5, 0);  // number[] (infiere del argumento)

// Infiere de multiples fuentes
function merge<T extends object, U extends object>(a: T, b: U): T & U {
  return { ...a, ...b };
}

const combinado = merge({ nombre: "Andres" }, { edad: 30 });
// { nombre: string } & { edad: number }
// = { nombre: string; edad: number }
```

---

## 11.6 Conditional Types con Generics

```typescript
// Tipo que depende de una condicion
type EsArray<T> = T extends unknown[] ? true : false;

type A = EsArray<string[]>;   // true
type B = EsArray<number>;     // false

// Extraer tipo de elemento de array
type Elemento<T> = T extends (infer E)[] ? E : never;
type E1 = Elemento<number[]>;      // number
type E2 = Elemento<{ id: number }[]>; // { id: number }

// Extraer tipo de promesa (built-in: Awaited<T>)
type Desempaquetar<T> = T extends Promise<infer U> ? U : T;
type D1 = Desempaquetar<Promise<string>>;  // string
type D2 = Desempaquetar<number>;           // number (no es Promise)

// Tipos condicionales distribuidos (distributive conditional types)
type Diff<T, U> = T extends U ? never : T;
type SoloPrimitivos = Diff<"a" | "b" | 1 | 2, number>; // "a" | "b"
```

---

## 11.7 Cuando Usar (y No Usar) Generics

```typescript
// ✅ USAR generics cuando:
// 1. El codigo es identico para multiples tipos
function ultimoElemento<T>(arr: T[]): T | undefined {
  return arr[arr.length - 1];
}

// 2. Quieres relacionar tipos entre parametros
function asignar<T, K extends keyof T>(obj: T, key: K, value: T[K]): void {
  obj[key] = value;
}

// 3. Construyes estructuras de datos reutilizables
class Cola<T> { /* ... */ }

// ❌ NO USAR generics cuando:
// 1. Solo usas un tipo
// function sumar<T extends number>(a: T, b: T): T  // innecesario

// 2. La funcion ya acepta "any" de forma natural
// function imprimir<T>(valor: T): void  // usa unknown o el tipo real

// 3. Generan mas complejidad que beneficio
// type SuperComplejo<T extends ...> = T extends ... ? ... : ...
// Si necesitas 5 minutos para entenderlo, simplificalo
```

---

## 11.8 Variance Annotations (in/out, TypeScript 4.7+)

```typescript
// Variance: controla el subtipado de tipos genericos

// Invariante (por defecto)
interface Repository<T> {
  get(id: string): T;       // T en posicion de salida (covariante)
  save(item: T): void;      // T en posicion de entrada (contravariante)
}
// Repository<Admin> NO es subtipo de Repository<Usuario>

// out (covariant): solo posiciones de salida, permite subtipado
interface ReadOnlyRepo<out T> {
  get(id: string): T;
  list(): T[];
  // save(item: T): void; // ❌ Error: out no permite entrada
}

class Admin extends Usuario {}
declare const adminRepo: ReadOnlyRepo<Admin>;
const userRepo: ReadOnlyRepo<Usuario> = adminRepo; // ✅ OK

// in (contravariant): solo posiciones de entrada
interface WriteOnlyRepo<in T> {
  save(item: T): void;
  delete(item: T): void;
  // get(id: string): T; // ❌ Error: in no permite salida
}

declare const userWrite: WriteOnlyRepo<Usuario>;
const adminWrite: WriteOnlyRepo<Admin> = userWrite; // ✅ OK (al reves!)
```

### noInfer (TypeScript 5.4+)

```typescript
// noInfer: bloquea la inferencia desde ese argumento
declare function crearAnimal<T>(nombre: T, crear: (nombre: string) => T): T;

// Sin noInfer: T se infiere del primer argumento (string)
const perro = crearAnimal("Firulais", (n) => ({ nombre: n, raza: "Perro" }));
// T = string, pero createAnimal retorna { nombre, raza } -> conflicto

declare function crearAnimal<const T>(
  nombre: NoInfer<T>,
  crear: (nombre: string) => T,
): T;

// Con noInfer: T solo se infiere del callback
// const gato = crearAnimal("Michi", (n) => ({ nombre: n, tipo: "gato" }));
// T = { nombre: string; tipo: string }
```

### Generic Default Parameters

```typescript
// Valor por defecto en type params
interface Resultado<T, E = Error> {
  ok: boolean;
  value?: T;
  error?: E;
}

// E es Error por defecto, no necesitas especificarlo
const r1: Resultado<number> = { ok: true, value: 42 };
const r2: Resultado<number, string> = { ok: false, error: "fallo" };

// Utility type con generics por defecto
type Entorno<T extends string = "development" | "production"> = {
  modo: T;
  debug: T extends "development" ? true : false;
};

const dev: Entorno = { modo: "development", debug: true }; // Entorno<"development">
```

---

## 11.9 ConstructorParameters e InstanceType

```typescript
class Usuario {
  constructor(
    public id: number,
    public nombre: string,
  ) {}
}

// Obtener tipos de parametros del constructor
type ParametrosUsuario = ConstructorParameters<typeof Usuario>;
// [id: number, nombre: string]

// Obtener tipo de instancia de una clase
type TipoUsuario = InstanceType<typeof Usuario>;
// Usuario

// Factory generica type-safe
function crear<T extends abstract new (...args: any[]) => any>(
  Clase: T,
  ...args: ConstructorParameters<T>
): InstanceType<T> {
  return new Clase(...args);
}

const usuario = crear(Usuario, 1, "Andres"); // Usuario, type-safe
```

---

## Resumen del Capítulo

- Generics (`<T>`) permiten codigo reutilizable type-safe con inferencia automatica.
- Constraints (`<T extends Interface>`) limitan los tipos aceptados.
- `keyof T` obtiene las claves de un tipo como union de literales.
- Utility types (`Partial<T>`, `Pick<T, K>`, `Record<K, V>`) son generics predefinidos.
- Conditional types (`T extends U ? X : Y`) crean tipos condicionales con `infer`.
- Patrones: Builder type-safe, Repository generico, Result type.
- Usa generics cuando el codigo es identico para multiples tipos y quieres relacionarlos.
- Evita generics cuando añaden complejidad sin beneficio real.

En el siguiente capitulo exploraremos las buenas practicas y el TypeScript idiomatico.

---

← [Capítulo anterior](10-testing.md) | [Inicio](README.md) | [Capítulo siguiente →](12-buenas-practicas.md)
