# Capítulo 5: Estructuras de Datos

TypeScript enriquece las estructuras de datos nativas de JavaScript con tipos estaticos, permitiendo modelar colecciones con precision. Este capitulo cubre arrays, objetos, Map, Set, clases y tipos de utilidad.

---

## 5.1 Arrays

### Declaracion y tipado

```typescript
// Sintaxis con []
const numeros: number[] = [1, 2, 3, 4, 5];
const nombres: string[] = ["Ana", "Luis", "Maria"];

// Sintaxis generica con Array<T>
const booleanos: Array<boolean> = [true, false, true];

// Arrays de tipos complejos
interface Usuario {
  id: number;
  nombre: string;
  email: string;
}

const usuarios: Usuario[] = [
  { id: 1, nombre: "Andres", email: "a@test.com" },
  { id: 2, nombre: "Maria", email: "m@test.com" },
];

// Arrays de solo lectura
const inmutable: readonly number[] = [1, 2, 3];
// inmutable.push(4);   // ❌ Error
// inmutable[0] = 10;   // ❌ Error

// ReadonlyArray<T>
const tambienInmutable: ReadonlyArray<string> = ["a", "b", "c"];
```

### Metodos funcionales en profundidad

```typescript
const datos = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10];

// map: transformar cada elemento
const dobles = datos.map((n: number): number => n * 2);

// filter: filtrar elementos
const pares = datos.filter((n: number): boolean => n % 2 === 0);

// reduce: acumular en un solo valor
const suma = datos.reduce((acumulador: number, actual: number): number => {
  return acumulador + actual;
}, 0);

// reduce con tipos de retorno diferentes al tipo del array
interface Estadisticas {
  total: number;
  promedio: number;
  max: number;
  min: number;
}

const stats: Estadisticas = datos.reduce(
  (acc, n) => ({
    total: acc.total + n,
    promedio: (acc.total + n) / (datos.length),
    max: Math.max(acc.max, n),
    min: Math.min(acc.min, n),
  }),
  { total: 0, promedio: 0, max: -Infinity, min: Infinity },
);

// flatMap: map + flat
const duplicados = datos.flatMap((n) => [n, n * 2]);
// [1, 2, 2, 4, 3, 6, ...]

// sort con comparador tipado
usuarios.sort((a: Usuario, b: Usuario): number => a.nombre.localeCompare(b.nombre));

// Array.from con mapeo
const cuadrados = Array.from({ length: 10 }, (_, i) => (i + 1) ** 2);
// [1, 4, 9, 16, ...]
```

### Patrones con arrays

```typescript
// Agrupar por clave (groupBy manual)
function agruparPor<T, K extends string | number | symbol>(
  items: T[],
  selector: (item: T) => K,
): Record<K, T[]> {
  return items.reduce(
    (grupos, item) => {
      const clave = selector(item);
      (grupos[clave] ??= []).push(item);
      return grupos;
    },
    {} as Record<K, T[]>,
  );
}

const porEdad = agruparPor(usuarios, (u) => (u.edad >= 18 ? "adulto" : "menor"));

// Object.groupBy (ES2024 / TypeScript 5.4+)
// const agrupados = Object.groupBy(usuarios, (u) => u.edad >= 18 ? "adulto" : "menor");

// Tomar hasta N elementos que cumplan condicion
function tomarMientras<T>(items: T[], predicado: (item: T) => boolean): T[] {
  const resultado: T[] = [];
  for (const item of items) {
    if (!predicado(item)) break;
    resultado.push(item);
  }
  return resultado;
}

// Particionar array en dos segun predicado
function particionar<T>(items: T[], predicado: (item: T) => boolean): [T[], T[]] {
  return items.reduce(
    ([si, no], item) => {
      (predicado(item) ? si : no).push(item);
      return [si, no];
    },
    [[], []] as [T[], T[]],
  );
}

const [paresArray, imparesArray] = particionar(datos, (n) => n % 2 === 0);
```

---

## 5.2 Objetos y Tipos de Objeto

### Objetos literales con tipos

```typescript
// Tipo inferido
const usuario = {
  id: 1,
  nombre: "Andres",
  email: "andres@ejemplo.com",
};
// Tipo inferido: { id: number; nombre: string; email: string }

// Tipo explicito con interfaz
interface Usuario {
  id: number;
  nombre: string;
  email: string;
  edad?: number;
}

const usuario2: Usuario = {
  id: 2,
  nombre: "Maria",
  email: "maria@ejemplo.com",
};

// Index signatures: objetos con claves dinamicas
interface Diccionario<T> {
  [clave: string]: T;
}

const colores: Diccionario<string> = {
  rojo: "#FF0000",
  verde: "#00FF00",
  azul: "#0000FF",
};

// Record utility type (equivalente a index signature)
const precios: Record<string, number> = {
  manzana: 1.5,
  banana: 0.8,
  cereza: 3.0,
};
```

### Metodos utiles de Object

```typescript
const config: Usuario = {
  id: 1,
  nombre: "Andres",
  email: "a@test.com",
};

// Object.keys: claves como string[] (limitacion de TypeScript)
const claves = Object.keys(config); // string[]

// Solucion tipada:
const clavesTipadas = Object.keys(config) as Array<keyof Usuario>;
// ("id" | "nombre" | "email" | "edad")[]

// Object.entries: pares [clave, valor] tipados
for (const [clave, valor] of Object.entries(config)) {
  console.log(`${clave}: ${valor}`);
}

// Object.fromEntries: crear objeto desde array de pares
const pares: Array<[string, number]> = [
  ["a", 1],
  ["b", 2],
  ["c", 3],
];
const objeto = Object.fromEntries(pares); // { a: 1, b: 2, c: 3 }

// Object.freeze: objeto inmutable en runtime + readonly en tipos
const CONFIG = Object.freeze({
  puerto: 8080,
  host: "localhost",
} as const);
```

---

## 5.3 Maps y Sets

### Map<K, V>

```typescript
// Map: coleccion clave-valor con cualquier tipo de clave
const cache = new Map<string, Usuario>();

cache.set("user-1", { id: 1, nombre: "Andres", email: "a@test.com" });
cache.set("user-2", { id: 2, nombre: "Maria", email: "m@test.com" });

const usuario = cache.get("user-1"); // Usuario | undefined
console.log(cache.has("user-3"));    // false
console.log(cache.size);             // 2

// Iterar Map con tipos completos
for (const [clave, valor] of cache) {
  console.log(`${clave}: ${valor.nombre}`);
}

// Map con claves de objeto
const metadata = new Map<object, string>();
const obj1 = { id: 1 };
metadata.set(obj1, "creado el 2024-01-01");

// Convertir Map a array de pares
const pares = [...cache.entries()]; // Array<[string, Usuario]>
```

### Map vs Object: cuando usar cada uno

| Caracteristica | Object | Map |
|---------------|--------|-----|
| Tipo de clave | string o symbol | Cualquier tipo |
| Orden de insercion | Generalmente si (ES2015+) | Garantizado |
| Tamaño | `Object.keys(obj).length` | `map.size` |
| Iteracion | `Object.entries()` | `for...of`, `.forEach()` |
| Rendimiento (muchas inserciones/eliminaciones) | Mas lento | Mas rapido |
| Serializacion JSON | Nativa | No nativa |

### Set<T>

```typescript
// Set: coleccion de valores unicos
const tags = new Set<string>();

tags.add("typescript");
tags.add("nodejs");
tags.add("typescript"); // Ignorado: ya existe

console.log(tags.size);    // 2
console.log(tags.has("typescript")); // true

tags.delete("nodejs");

// Iterar Set
for (const tag of tags) {
  console.log(tag);
}

// Operaciones de conjuntos con Set
const setA = new Set([1, 2, 3, 4]);
const setB = new Set([3, 4, 5, 6]);

// Union
const union = new Set([...setA, ...setB]); // {1,2,3,4,5,6}

// Interseccion
const interseccion = new Set([...setA].filter((x) => setB.has(x))); // {3,4}

// Diferencia
const diferencia = new Set([...setA].filter((x) => !setB.has(x))); // {1,2}

// Eliminar duplicados de un array
const conDuplicados = [1, 2, 2, 3, 3, 3, 4];
const unicos = [...new Set(conDuplicados)]; // [1, 2, 3, 4]
```

---

## 5.4 Tuplas

```typescript
// Tupla: array con tipos fijos por posicion
type Punto2D = [number, number];
type Punto3D = [number, number, number];
type RespuestaHTTP = [codigo: number, mensaje: string];

const origen: Punto2D = [0, 0];
const respuesta: RespuestaHTTP = [200, "OK"];

// Acceso tipado por indice
const x: number = origen[0];
const y: number = origen[1];

// Desestructuracion con tipos
const [codigo, mensaje] = respuesta;

// Tupla con elementos opcionales y rest
type EntradaCSV = [id: number, nombre: string, ...tags: string[]];
const fila1: EntradaCSV = [1, "Andres", "admin", "activo"];
const fila2: EntradaCSV = [2, "Maria"]; // OK: tags opcional

// Tuplas de solo lectura
const CONSTANTES = [Math.PI, Math.E, 299_792_458] as const;
// Tipo: readonly [3.14159..., 2.71828..., 299792458]
```

---

## 5.5 Clases

### Clase basica con tipos

```typescript
class Usuario {
  // Propiedades con tipos (strictPropertyInitialization requiere inicializacion)
  readonly id: number;
  nombre: string;
  private _email: string;
  protected rol: "admin" | "usuario" = "usuario";

  constructor(id: number, nombre: string, email: string) {
    this.id = id;
    this.nombre = nombre;
    this._email = email;
  }

  // Getter (propiedad calculada)
  get email(): string {
    return this._email.toLowerCase();
  }

  // Setter (validacion en asignacion)
  set email(valor: string) {
    if (!valor.includes("@")) {
      throw new Error("Email invalido");
    }
    this._email = valor;
  }

  // Metodo publico
  saludar(): string {
    return `Hola, soy ${this.nombre}`;
  }

  // Metodo privado (solo dentro de la clase)
  private validarEmail(email: string): boolean {
    return email.includes("@");
  }

  // Metodo estatico (no necesita instancia)
  static crearAnonimo(): Usuario {
    return new Usuario(0, "Anonimo", "anonimo@test.com");
  }
}

const usuario = new Usuario(1, "Andres", "Andres@EJEMPLO.com");
console.log(usuario.email);     // "andres@ejemplo.com"
console.log(usuario.saludar()); // "Hola, soy Andres"

const anonimo = Usuario.crearAnonimo();
```

### Herencia y override

```typescript
class Admin extends Usuario {
  private nivel: number;

  constructor(id: number, nombre: string, email: string, nivel: number) {
    super(id, nombre, email);  // Llamar al constructor padre
    this.nivel = nivel;
    this.rol = "admin";        // Acceder a propiedad protected
  }

  // Override de metodo
  override saludar(): string {
    return `${super.saludar()} (Admin nivel ${this.nivel})`;
  }

  // Metodo exclusivo de Admin
  banearUsuario(usuario: Usuario): void {
    console.log(`${usuario.nombre} ha sido baneado`);
  }
}

const admin = new Admin(2, "SuperAdmin", "admin@test.com", 3);
console.log(admin.saludar()); // "Hola, soy SuperAdmin (Admin nivel 3)"
```

### Clases con generics

```typescript
class Pila<T> {
  private elementos: T[] = [];

  push(item: T): void {
    this.elementos.push(item);
  }

  pop(): T | undefined {
    return this.elementos.pop();
  }

  get cima(): T | undefined {
    return this.elementos[this.elementos.length - 1];
  }

  get tamano(): number {
    return this.elementos.length;
  }

  get vacia(): boolean {
    return this.elementos.length === 0;
  }

  [Symbol.iterator](): Iterator<T> {
    return this.elementos[Symbol.iterator]();
  }
}

const pilaNumeros = new Pila<number>();
pilaNumeros.push(1);
pilaNumeros.push(2);
console.log(pilaNumeros.pop()); // 2
```

---

## 5.6 Utility Types para Estructuras de Datos

```typescript
interface Usuario {
  id: number;
  nombre: string;
  email: string;
  edad: number;
  activo: boolean;
}

// Partial<T>: todas las propiedades opcionales
type UsuarioParcial = Partial<Usuario>;
// { id?: number; nombre?: string; ... }

function actualizarUsuario(id: number, datos: Partial<Usuario>): void {
  // Solo pasas los campos que cambian
}
actualizarUsuario(1, { nombre: "Nuevo Nombre" });

// Required<T>: todas las propiedades obligatorias
type UsuarioCompleto = Required<Usuario>;

// Readonly<T>: todas las propiedades readonly
type UsuarioInmutable = Readonly<Usuario>;

// Pick<T, K>: seleccionar propiedades
type UsuarioBasico = Pick<Usuario, "id" | "nombre" | "email">;

// Omit<T, K>: excluir propiedades
type UsuarioSinID = Omit<Usuario, "id">;

// Record<K, V>: objeto con claves K y valores V
type UsuariosPorID = Record<number, Usuario>;
// { [id: number]: Usuario }

const cache: UsuariosPorID = {
  1: { id: 1, nombre: "Andres", email: "a@test.com", edad: 30, activo: true },
  2: { id: 2, nombre: "Maria", email: "m@test.com", edad: 25, activo: false },
};
```

---

## 5.7 WeakMap y WeakSet

> [!NOTE]
> ### 🎈 Los Post-It Adhesivos en Globos de Helio (WeakMap vs. Map)
>
> Imagina que en JavaScript los **objetos** que creas son **globos de helio flotantes**. Mientras los tengas atados de un hilo en tu mano (referencias al objeto), los globos se mantienen flotando en la habitación. Si sueltas el hilo por completo, el globo vuela por la ventana hacia el cielo y desaparece para siempre (el recolector de basura o Garbage Collector lo libera de la memoria).
>
> - **El Map Convencional (El Gancho Metálico)**: Si usas un objeto globo como clave en un `Map` convencional, el mapa le ata un **cable de acero pesado** al globo. Aunque sueltes el hilo de tu mano, el globo nunca se irá volando porque el mapa lo tiene anclado a la fuerza en su catálogo. Si olvidas eliminarlo del mapa (`map.delete(objeto)`), el globo se quedará ahí atrapado para siempre, causando una **fuga de memoria (memory leak)**.
> - **El WeakMap (El Post-It en el Globo)**: Un `WeakMap` no ata ningún cable de acero. Es como escribir información en un **pequeño papel Post-It adhesivo y pegarlo directamente en la superficie del globo**.
>   - Mientras tengas el hilo del globo en la mano, puedes leer el papel adhesivo (`weakmap.get(globo)`).
>   - En cuanto sueltas el hilo y el globo sale volando por la ventana, **el Post-It se va adherido al globo hacia el cielo**. El recolector de basura destruye el globo y el Post-It al mismo tiempo, limpiando la memoria de forma totalmente automática y sin tu intervención.
>
> **En resumen**: Las claves de un `WeakMap` son exclusivamente objetos mantenidos con *referencias débiles*. Si pierdes la referencia principal del objeto clave en tu aplicación, el sistema liberará la memoria de inmediato y limpiará el WeakMap de manera transparente.

```typescript
// WeakMap: claves debiles (objetos), no previene garbage collection
// IDEAL para: metadata privada, caches, datos asociados a objetos

const metadata = new WeakMap<object, { creadoEn: Date; contador: number }>();

class Usuario {
  constructor(public nombre: string) {}
}

const user = new Usuario("Andres");

metadata.set(user, { creadoEn: new Date(), contador: 0 });

// Recuperar metadata
const meta = metadata.get(user)!;
meta.contador++;
console.log(meta.creadoEn);

// Cuando user ya no es referenciado, el GC puede liberar la entrada
// del WeakMap automaticamente (no hay fuga de memoria)

// WeakSet: coleccion de objetos unicos con referencias debiles
const usuariosActivos = new WeakSet<Usuario>();

usuariosActivos.add(user);
console.log(usuariosActivos.has(user)); // true
```

### Caso practico: contador de visitas por objeto

```typescript
const visitasPorPagina = new WeakMap<object, number>();

function visitar(pagina: object): number {
  const actual = visitasPorPagina.get(pagina) ?? 0;
  visitasPorPagina.set(pagina, actual + 1);
  return actual + 1;
}

const pagina1 = { ruta: "/home" };
console.log(visitar(pagina1)); // 1
console.log(visitar(pagina1)); // 2
```

---

## 5.8 Proxy y Reflect

```typescript
// Proxy: intercepta operaciones sobre objetos
const handler: ProxyHandler<Record<string, number>> = {
  get(target, prop) {
    console.log(`Leyendo ${String(prop)}`);
    return target[prop as string] ?? 0; // Valor por defecto
  },
  set(target, prop, value) {
    if (typeof value !== "number") {
      throw new Error(`Solo se permiten numeros, recibido: ${typeof value}`);
    }
    console.log(`Asignando ${String(prop)} = ${value}`);
    target[prop as string] = value;
    return true;
  },
};

const contadores = new Proxy({} as Record<string, number>, handler);
contadores.visitas = 10;  // Asignando visitas = 10
console.log(contadores.visitas);  // Leyendo visitas -> 10
console.log(contadores.inexistente); // Leyendo inexistente -> 0
// contadores.visitas = "hola";  // ❌ Error: solo numeros
```

### Proxy para objetos reactivos

```typescript
type Listener<T> = (obj: T) => void;

function crearReactivo<T extends object>(
  obj: T,
  onChange: Listener<T>,
): T {
  return new Proxy(obj, {
    set(target, prop, value) {
      (target as any)[prop] = value;
      onChange(target);
      return true;
    },
    deleteProperty(target, prop) {
      delete (target as any)[prop];
      onChange(target);
      return true;
    },
  });
}

const estado = crearReactivo(
  { contador: 0, mensaje: "Hola" },
  (nuevoEstado) => console.log("Estado actualizado:", nuevoEstado),
);

estado.contador++;  // Estado actualizado: { contador: 1, mensaje: "Hola" }
```

---

## 5.9 Datos Binarios: Buffer, TypedArray, ArrayBuffer

```typescript
// Buffer: Node.js para datos binarios (herencia de Uint8Array)
const buf = Buffer.from("Hola, mundo!", "utf-8");
console.log(buf);                 // <Buffer 48 6f 6c 61 2c 20 6d 75 ...>
console.log(buf.toString("hex")); // "486f6c612c206d756e646f21"
console.log(buf.toString("base64")); // "SG9sYSwgbXVuZG8h"

// Crear y manipular buffers
const header = Buffer.alloc(16);    // 16 bytes en cero
header.writeUInt32BE(42, 0);        // Escribir entero big-endian
header.writeUInt32BE(100, 4);
console.log(header.readUInt32BE(0)); // 42

// TypedArray: arrays tipados para datos binarios (JavaScript estandar)
const int32 = new Int32Array([1, 2, 3, 4]);
const uint8 = new Uint8Array(int32.buffer);
console.log(uint8); // Representacion byte a byte

// ArrayBuffer: buffer de tamaño fijo
const buffer = new ArrayBuffer(16);
const vista = new DataView(buffer);
vista.setInt32(0, 42, true);  // little-endian
vista.setFloat64(4, 3.14159);
console.log(vista.getInt32(0, true));   // 42
console.log(vista.getFloat64(4));       // 3.14159

// structuredClone: clonacion profunda (Node 17+)
const original = {
  fecha: new Date(),
  mapa: new Map([["a", 1]]),
  set: new Set([1, 2, 3]),
  buffer: Buffer.from("hola"),
};
const copia = structuredClone(original);
copia.mapa.set("b", 2);
console.log(original.mapa.has("b")); // false (copia independiente)
```

---

## 5.10 Intl y URL APIs

```typescript
// Intl: internacionalizacion nativa
const formatoMoneda = new Intl.NumberFormat("es-MX", {
  style: "currency",
  currency: "MXN",
});
console.log(formatoMoneda.format(1234567.89)); // "$1,234,567.89"

const formatoFecha = new Intl.DateTimeFormat("es-ES", {
  dateStyle: "full",
  timeStyle: "long",
});
console.log(formatoFecha.format(new Date())); // "martes, 15 de enero de 2024, 10:30:00 CET"

const formatoRelativo = new Intl.RelativeTimeFormat("es", { numeric: "auto" });
console.log(formatoRelativo.format(-3, "day"));  // "hace 3 dias"
console.log(formatoRelativo.format(2, "hour"));  // "dentro de 2 horas"

// URL: parseo y manipulacion segura
const url = new URL("https://api.ejemplo.com/usuarios?pagina=1&limite=10#seccion");

console.log(url.protocol);    // "https:"
console.log(url.hostname);    // "api.ejemplo.com"
console.log(url.pathname);    // "/usuarios"
console.log(url.searchParams.get("pagina"));  // "1"
console.log(url.hash);        // "#seccion"

// Construir URLs de forma segura
const base = new URL("https://api.ejemplo.com");
base.pathname = "/v2/productos";
base.searchParams.set("categoria", "electronica");
base.searchParams.set("orden", "precio");
console.log(base.toString());
// "https://api.ejemplo.com/v2/productos?categoria=electronica&orden=precio"

// URLSearchParams: manejar query strings
const params = new URLSearchParams("q=typescript&lang=es&lang=en");
console.log(params.get("q"));        // "typescript"
console.log(params.getAll("lang"));  // ["es", "en"]
params.append("lang", "fr");
params.delete("q");
console.log(params.toString());      // "lang=es&lang=en&lang=fr"
```

---

## Resumen del Capítulo

- Arrays: usa `T[]` o `Array<T>`. Metodos funcionales (`map`, `filter`, `reduce`) como primera opcion.
- Objetos: interfaces y `Record<K,V>` para diccionarios. `Object.entries()` para iterar.
- `Map<K,V>` para claves no-string y alto rendimiento en inserciones; `Set<T>` para valores unicos.
- Tuplas (`[number, string]`) para arrays con tipos por posicion.
- Clases con `private`, `protected`, `public`, `readonly`, getters/setters y herencia.
- Utility types (`Partial`, `Pick`, `Omit`, `Record`) reutilizan y transforman tipos existentes.
- `as const` crea tipos literales readonly inmutables.
- Prefiere interfaces para objetos, type aliases para unions y tuplas.

En el siguiente capitulo exploraremos las interfaces y tipos avanzados en TypeScript.

---

← [Capítulo anterior](04-funciones.md) | [Inicio](README.md) | [Capítulo siguiente →](06-interfaces.md)
