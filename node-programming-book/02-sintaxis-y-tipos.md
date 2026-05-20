# Capítulo 2: Sintaxis y Tipos en TypeScript

TypeScript extiende JavaScript con un sistema de tipos rico y expresivo. Este capitulo cubre los tipos fundamentales, como declararlos y como TypeScript infiere y verifica tipos automaticamente.

---

## 2.1 Variables y su Alcance

### let, const y var

```typescript
// const: valor inmutable (no se puede reasignar)
const PI = 3.14159;
const usuario = { nombre: "Andres" };
usuario.nombre = "Carlos";  // OK: el objeto es mutable, la referencia es constante
// usuario = {};             // ❌ Error: no se puede reasignar const

// let: mutable, con block scope
let contador = 0;
contador = 1;                // OK

// var: EVITAR. Tiene function scope y hoisting (legacy)
// var globalScope = "peligroso"; // No uses var en TypeScript moderno
```

### Block scope vs function scope

```typescript
function ejemploScope() {
  if (true) {
    const bloque = "solo visible aqui";
    let tambienBloque = "tampoco sale";
    var functionScope = "visible en toda la funcion"; // ⚠️ EVITAR
  }

  // console.log(bloque);          // ❌ Error: bloque no existe aqui
  console.log(functionScope);      // OK: var ignora el bloque (wtf!)
}

// TypeScript moderno: usa const por defecto, let solo si necesitas mutar.
```

### const assertions y objetos inmutables

```typescript
// as const: hace que el tipo sea el literal exacto, no el tipo general
const colores = ["rojo", "verde", "azul"] as const;
// Tipo inferido: readonly ["rojo", "verde", "azul"]

const config = {
  puerto: 8080,
  host: "localhost",
  debug: true,
} as const;
// Tipo inferido: { readonly puerto: 8080; readonly host: "localhost"; readonly debug: true }

// config.puerto = 9090;  // ❌ Error: readonly
```

---

## 2.2 Tipos Basicos

### number

```typescript
const entero: number = 42;
const decimal: number = 3.14;
const hexadecimal: number = 0xff;  // 255
const binario: number = 0b1010;     // 10
const octal: number = 0o744;        // 484

// Cuidado con NaN e Infinity
const noEsNumero: number = NaN;
const infinito: number = Infinity;

// Verificar NaN (NaN !== NaN, usar Number.isNaN)
console.log(Number.isNaN(NaN));     // true
console.log(Number.isNaN(42));      // false
console.log(Number.isFinite(1/0));  // false
```

### string

```typescript
const simple: string = 'Hola';
const doble: string = "Mundo";
const template: string = `Hola ${simple}, tienes ${30 + 5} años`;

// Metodos comunes con tipos
const texto: string = "  Node.js y TypeScript  ";
console.log(texto.trim());                  // "Node.js y TypeScript"
console.log(texto.toUpperCase());           // "  NODE.JS Y TYPESCRIPT  "
console.log(texto.includes("TypeScript"));  // true
console.log(texto.split(" "));              // ["", "", "Node.js", "y", "TypeScript", "", ""]
console.log(texto.replace("TypeScript", "TS")); // "  Node.js y TS  "

// Template literals con tipos seguros
const nombre: string = "Andres";
const edad: number = 30;
const presentacion = `${nombre} tiene ${edad} años`;
```

### boolean

```typescript
const verdadero: boolean = true;
const falso: boolean = false;

// Operadores logicos
const resultado = verdadero && falso;  // false (AND)
const cualquiera = verdadero || falso; // true  (OR)
const negado = !verdadero;             // false (NOT)

// Truthy/Falsy en JavaScript (TypeScript no cambia esto)
// Falsy: false, 0, "", null, undefined, NaN
// Truthy: todo lo demas

// Type narrowing con booleanos
function procesar(valor: string | null): string {
  if (valor) {
    // TypeScript sabe que aqui valor es string (no null)
    return valor.toUpperCase();
  }
  return "VALOR NULO";
}
```

### null y undefined

```typescript
// Con strictNullChecks: true (recomendado), null/undefined son tipos distintos
const nulo: null = null;
const indefinido: undefined = undefined;

// Opcional: union con undefined
let opcional: string | undefined;
opcional = "valor";
opcional = undefined;  // OK
// opcional = null;    // Con strictNullChecks, null no es valido si solo pusiste undefined

// Optional chaining (?.)
interface Direccion {
  calle: string;
  ciudad?: {
    nombre: string;
    codigoPostal?: number;
  };
}

const direccion: Direccion = { calle: "Principal" };
console.log(direccion.ciudad?.nombre);        // undefined (sin error)
console.log(direccion.ciudad?.codigoPostal);  // undefined

// Nullish coalescing (??)
const puerto = process.env.PORT ?? 8080;  // 8080 si PORT es null o undefined
const puerto2 = process.env.PORT || 8080; // 8080 si PORT es falsy (incluye "")
// Diferencia: ?? solo salta en null/undefined, || salta en cualquier falsy
```

### symbol

```typescript
// Symbols: identificadores unicos e inmutables
const sym1: symbol = Symbol("descripcion");
const sym2: symbol = Symbol("descripcion");
console.log(sym1 === sym2);  // false (cada Symbol es unico)

// Simbolos globales
const GLOBAL_KEY = Symbol.for("app.key");
const mismoSimbolo = Symbol.for("app.key");
console.log(GLOBAL_KEY === mismoSimbolo); // true

// Casos de uso: claves privadas, constantes unicas
const LOG_LEVEL = {
  DEBUG: Symbol("DEBUG"),
  INFO: Symbol("INFO"),
  ERROR: Symbol("ERROR"),
} as const;
```

### bigint

```typescript
// BigInt: enteros de precision arbitraria
const numeroGrande: bigint = 9007199254740991n;
const desdeNumero: bigint = BigInt(42);
const sumaBigInt = numeroGrande + 1n;

// No se pueden mezclar bigint con number
// const mezcla = numeroGrande + 1;  // ❌ Error
```

---

## 2.3 Tipos Literales y Union Types

### Tipos literales

```typescript
// Un tipo puede ser un valor especifico, no solo una categoria
type Direccion = "norte" | "sur" | "este" | "oeste";
type MetodoHTTP = "GET" | "POST" | "PUT" | "DELETE" | "PATCH";
type CodigoEstado = 200 | 201 | 400 | 401 | 403 | 404 | 500;
type Verdadero = true;  // Solo acepta true

function mover(direccion: Direccion): void {
  console.log(`Moviendo hacia ${direccion}`);
}

mover("norte");   // OK
// mover("arriba"); // ❌ Error: no es una direccion valida

// Combinar tipos literales con otros tipos
type ResultadoAPI =
  | { estado: "exito"; datos: unknown }
  | { estado: "error"; mensaje: string; codigo: number };
```

### Union types

```typescript
// Una variable puede aceptar multiples tipos
type ID = string | number;

function buscarPorID(id: ID): void {
  if (typeof id === "string") {
    console.log(`Buscando por string: ${id.toUpperCase()}`);
  } else {
    console.log(`Buscando por numero: ${id.toFixed(0)}`);
  }
}

buscarPorID(123);      // OK
buscarPorID("abc");    // OK
// buscarPorID(true);  // ❌ Error

// Unions de objetos discriminados (discriminated unions)
type Exito = { tipo: "exito"; datos: string };
type Error = { tipo: "error"; mensaje: string };
type Cargando = { tipo: "cargando" };

type EstadoPeticion = Exito | Error | Cargando;

function manejarEstado(estado: EstadoPeticion): string {
  switch (estado.tipo) {
    case "exito":    return `Datos: ${estado.datos}`;
    case "error":    return `Error: ${estado.mensaje}`;
    case "cargando": return "Cargando...";
  }
}
```

### Intersection types

```typescript
// Combina multiples tipos en uno solo
interface ConNombre {
  nombre: string;
}

interface ConEdad {
  edad: number;
}

interface ConEmail {
  email: string;
}

type Persona = ConNombre & ConEdad & ConEmail;

const persona: Persona = {
  nombre: "Andres",
  edad: 30,
  email: "andres@ejemplo.com",
};

// Intersection vs Union visualmente:
// Union (|): A | B = propiedades de A O de B (uno u otro)
// Intersection (&): A & B = propiedades de A Y de B (ambas)
```

---

## 2.4 Type Assertions y Type Narrowing

### Type assertions (as)

```typescript
// Le dices al compilador "confia en mi, se que tipo es"
const elemento = document.getElementById("app") as HTMLDivElement;
const valor = JSON.parse('{"nombre": "Andres"}') as { nombre: string };

// Sintaxis alternativa (evitar en JSX/TSX por conflicto con XML)
const elemento2 = <HTMLDivElement>document.getElementById("app");

// ASERCIÓN DOBLE (peligrosa, solo para casos extremos)
// Convierte a unknown primero, luego al tipo deseado
const datoDesconocido: unknown = "hola";
const comoNumero = datoDesconocido as unknown as number; // ⚠️ No hay verificacion real
```

### Type narrowing (refinamiento de tipos)

```typescript
// typeof: refinar tipos primitivos
function procesar(valor: string | number): string {
  if (typeof valor === "string") {
    return valor.toUpperCase();  // Aqui valor es string
  }
  return valor.toFixed(2);       // Aqui valor es number
}

// instanceof: refinar clases
class Usuario {
  constructor(public nombre: string) {}
}

class Admin {
  constructor(public nombre: string, public nivel: number) {}
}

function obtenerPermisos(entidad: Usuario | Admin): string[] {
  if (entidad instanceof Admin) {
    return ["leer", "escribir", "eliminar"];  // Admin tiene permisos totales
  }
  return ["leer"];  // Usuario solo lectura
}

// in operator: comprobar propiedades
interface Perro {
  ladrar(): void;
  raza: string;
}

interface Pajaro {
  volar(): void;
  envergadura: number;
}

function interactuar(animal: Perro | Pajaro): void {
  if ("ladrar" in animal) {
    animal.ladrar();  // animal es Perro
  } else {
    animal.volar();   // animal es Pajaro
  }
}

// Type predicates (is): funciones que refinan tipos
function esString(valor: unknown): valor is string {
  return typeof valor === "string";
}

function procesarSeguro(valor: unknown): string {
  if (esString(valor)) {
    return valor.toUpperCase();  // TypeScript sabe que es string
  }
  return String(valor);
}
```

### Satisfies (TypeScript 4.9+)

```typescript
// satisfies: verifica que un valor cumple un tipo sin cambiar su tipo inferido
const config = {
  puerto: 8080,
  host: "localhost",
  debug: true,
} satisfies Record<string, string | number | boolean>;

// El tipo inferido sigue siendo { puerto: number; host: string; debug: boolean }
// Pero verificamos que cumple con Record<string, string | number | boolean>
console.log(config.puerto.toFixed(2));  // OK: puerto es number

// Sin satisfies, tendrias que elegir entre:
// 1. Anotacion explicita (pierdes los tipos literales)
// 2. Sin anotacion (no verificas que cumple el contrato)
```

---

## 2.5 Enums

### Enums numericos

```typescript
enum Direccion {
  Norte,      // 0
  Sur,        // 1
  Este,       // 2
  Oeste,      // 3
}

console.log(Direccion.Norte);  // 0
console.log(Direccion[0]);     // "Norte" (reverse mapping)

// Valores personalizados
enum CodigoEstado {
  OK = 200,
  Creado = 201,
  NoEncontrado = 404,
  ErrorServidor = 500,
}

// Auto-incremento desde un valor
enum DiaSemana {
  Lunes = 1,
  Martes,     // 2
  Miercoles,  // 3
  Jueves,     // 4
  Viernes,    // 5
}
```

### Enums de string (mas seguros)

```typescript
enum Entorno {
  Desarrollo = "development",
  Pruebas = "staging",
  Produccion = "production",
}

function conectar(entorno: Entorno): void {
  switch (entorno) {
    case Entorno.Desarrollo:
      console.log("Conectando a BD local...");
      break;
    case Entorno.Produccion:
      console.log("Conectando a BD produccion...");
      break;
  }
}

conectar(Entorno.Produccion);
```

### const enum (inline, sin codigo extra en runtime)

```typescript
const enum Colores {
  Rojo = "#FF0000",
  Verde = "#00FF00",
  Azul = "#0000FF",
}

// Se compila inline: no genera objeto en JS
const colorFavorito = Colores.Azul;
// JS generado: const colorFavorito = "#0000FF";
```

> **Recomendacion**: Prefiere union types de string sobre enums cuando sea posible. Son mas simples, no generan codigo extra y se integran mejor con el ecosistema.

```typescript
// Mejor que enum (mas TypeScript idiomatico)
type Entorno = "development" | "staging" | "production";

const conectar = (entorno: Entorno): void => {
  // mismo comportamiento, sin codigo JS extra
};
```

---

## 2.6 Arrays y Tuplas

### Arrays tipados

```typescript
const numeros: number[] = [1, 2, 3, 4, 5];
const nombres: Array<string> = ["Ana", "Luis"];  // Sintaxis generica

// Arrays mixtos con union types
const mixto: (string | number)[] = ["hola", 42, "mundo"];

// Arrays de solo lectura
const inmutable: readonly number[] = [1, 2, 3];
// inmutable.push(4);   // ❌ Error
// inmutable[0] = 10;   // ❌ Error

// Metodos funcionales con tipos
const dobles = numeros.map((n: number): number => n * 2);
const pares = numeros.filter((n: number): boolean => n % 2 === 0);
const suma = numeros.reduce((acc: number, n: number): number => acc + n, 0);
const encontrado = numeros.find((n: number): boolean => n > 3);

// Array destructuring con tipos
const [primero, segundo, ...resto]: number[] = numeros;
```

### Tuplas (arrays con longitud y tipos fijos)

```typescript
// Tupla: array con posiciones tipadas
type Coordenada = [number, number];
const origen: Coordenada = [0, 0];
const punto: Coordenada = [10, 20];

// Tupla con elementos de distintos tipos
type RespuestaAPI = [number, string, unknown];
const exito: RespuestaAPI = [200, "OK", { id: 1 }];
const [codigo, mensaje, datos] = exito;

// Tupla con etiquetas (TypeScript 4.0+)
type Usuario = [id: number, nombre: string, activo: boolean];
const usuario: Usuario = [1, "Andres", true];

// Las etiquetas solo ayudan en el editor, no fuerzan los tipos
// usuario[0] sigue siendo number, no "id"

// Tupla con elementos opcionales y rest
type CSVFila = [string, number, ...string[]];
const fila1: CSVFila = ["Andres", 30, "Madrid", "ESP"];
const fila2: CSVFila = ["Maria", 25];  // OK: los string extra son opcionales
```

---

## 2.7 unknown, never, void y Otros Tipos Clave

### unknown vs any

```typescript
// any: DESACTIVA el type checking. Peligroso.
const dato: any = "hola";
dato.toUpperCase();          // OK en compilacion, pero sin verificacion
dato.noExiste.metodo();      // OK en compilacion, CRASH en runtime
const num: number = dato;    // any se asigna a cualquier tipo (contagioso)

// unknown: SEGURO. Obliga a verificar el tipo antes de usarlo.
const seguro: unknown = "hola";
// seguro.toUpperCase();     // ❌ Error: unknown no tiene metodos
// const n: number = seguro;  // ❌ Error: unknown no se asigna a number

// Debes hacer type narrowing primero
if (typeof seguro === "string") {
  console.log(seguro.toUpperCase());  // ✅ OK: string en este scope
}
```

| Caracteristica | `any` | `unknown` |
|---------------|-------|-----------|
| Asignable a otros tipos | Si (contagioso) | No |
| Acceso a propiedades/metodos | Si (sin verificacion) | No |
| Type narrowing necesario | No | Si |
| Uso recomendado | Solo migraciones/legacy | Valores externos (API, JSON, user input) |

### never

```typescript
// never: representa valores que NUNCA ocurren
// Caso 1: funcion que lanza (nunca retorna)
function lanzarError(mensaje: string): never {
  throw new Error(mensaje);
}

// Caso 2: bucle infinito
function bucleInfinito(): never {
  while (true) {
    // nunca sale
  }
}

// Caso 3: exhaustiveness checking (EL MAS IMPORTANTE)
type Estado = "activo" | "inactivo" | "pendiente";

function manejarEstado(estado: Estado): string {
  switch (estado) {
    case "activo": return "OK";
    case "inactivo": return "NO";
    case "pendiente": return "...";
    default:
      // Si agregas un nuevo estado, TypeScript marca error aqui
      const _agotado: never = estado;
      return _agotado;
  }
}

// never en unions: desaparece
type SoloStrings = string | never;  // string (never se elimina)
```

### void vs undefined

```typescript
// void: la funcion no retorna nada util
function loggear(msg: string): void {
  console.log(msg);
  // return implicito = undefined
}

// Retorno explicito undefined NO requiere void
function opcional(): number | undefined {
  if (Math.random() > 0.5) return 42;
  return undefined; // OK
}

// En callbacks, void acepta cualquier retorno (conveniencia)
const fn: () => void = () => "ignorado"; // OK: string se descarta
```

### keyof type operator

```typescript
// keyof: obtiene las claves de un tipo como union de literales
interface Usuario {
  id: number;
  nombre: string;
  email: string;
}

type ClavesUsuario = keyof Usuario; // "id" | "nombre" | "email"

// Usado con generics para acceso type-safe
function obtener<T, K extends keyof T>(obj: T, clave: K): T[K] {
  return obj[clave];
}

const usuario: Usuario = { id: 1, nombre: "Andres", email: "a@test.com" };
const nombre = obtener(usuario, "nombre"); // string
// obtener(usuario, "apellido"); // ❌ Error: "apellido" no es keyof Usuario

// keyof con index signatures
type Diccionario = { [clave: string]: number };
type ClavesDic = keyof Diccionario; // string | number (number por index signature)
```

### typeof en contexto de tipos

```typescript
// typeof en runtime (JavaScript): devuelve string
console.log(typeof 42);       // "number"
console.log(typeof "hola");   // "string"

// typeof en tipos (TypeScript): extrae el tipo de una variable
const config = {
  puerto: 8080,
  host: "localhost",
  debug: true,
};

type Config = typeof config;
// { puerto: number; host: string; debug: boolean }

// Muy util con ReturnType
function crearUsuario(nombre: string, email: string) {
  return { nombre, email, creadoEn: new Date() };
}

type Usuario = ReturnType<typeof crearUsuario>;
// { nombre: string; email: string; creadoEn: Date }
```

### declare (ambient declarations)

```typescript
// declare: le dice a TypeScript que algo existe en runtime
// Util para variables globales, modulos sin tipos, etc.

// Variable global del entorno
declare const API_KEY: string;
declare const VERSION: string;

// Funcion global (ej: expuesta por un script en el HTML)
declare function trackEvent(nombre: string, datos?: Record<string, unknown>): void;

// Extender tipos de modulos (module augmentation)
declare module "express" {
  interface Request {
    usuarioId?: string;
    sesion?: { token: string; expira: Date };
  }
}

// Modulo sin tipos
declare module "*.jpg" {
  const src: string;
  export default src;
}

declare module "*.css" {
  const styles: Record<string, string>;
  export default styles;
}
```

---

## Resumen del Capítulo

- Usa `const` por defecto, `let` solo si necesitas mutacion. Evita `var`.
- TypeScript tiene tipos basicos: `number`, `string`, `boolean`, `null`, `undefined`, `symbol`, `bigint`.
- `unknown` es el tipo seguro para valores desconocidos. `any` desactiva verificacion (evitar).
- `never` para funciones que no retornan y exhaustiveness checking en switches.
- `keyof T` obtiene las claves de un tipo como union. Esencial con generics.
- `typeof` en contexto de tipos extrae el tipo de una variable en tiempo de compilacion.
- `declare` declara tipos para cosas que existen en runtime sin definicion TypeScript.
- Los tipos literales y union types (`"GET" | "POST"`) hacen el codigo mas expresivo y seguro.
- `satisfies` verifica que un valor cumple un tipo sin perder los tipos inferidos.
- Las tuplas son arrays con longitud y tipos fijos en cada posicion.
- Las discriminated unions permiten modelar estados complejos de forma segura.

En el siguiente capitulo exploraremos las estructuras de control en TypeScript.

---

← [Capítulo anterior](01-introduccion.md) | [Inicio](README.md) | [Capítulo siguiente →](03-estructuras-de-control.md)
