# Capítulo 4: Funciones

Las funciones en TypeScript heredan toda la flexibilidad de JavaScript (first-class, closures, arrow functions) y añaden tipado estatico: parametros tipados, retorno tipado, sobrecarga y funciones genericas.

---

## 4.1 Declaracion y Parametros

### Funcion basica con tipos

```typescript
// Declaracion clasica con tipos
function saludar(nombre: string): string {
  return `Hola, ${nombre}!`;
}

// Arrow function con tipos
const despedir = (nombre: string): string => {
  return `Adios, ${nombre}!`;
};

// Arrow function concisa (retorno implicito)
const sumar = (a: number, b: number): number => a + b;

// Sin retorno (void)
function loggear(mensaje: string): void {
  console.log(mensaje);
}

// Nunca retorna (never) - lanza excepcion o bucle infinito
function lanzarError(mensaje: string): never {
  throw new Error(mensaje);
}
```

### Parametros opcionales y por defecto

```typescript
// Parametro opcional con ?
function crearUsuario(
  nombre: string,
  email: string,
  edad?: number,       // Opcional: number | undefined
): Usuario {
  return {
    nombre,
    email,
    edad: edad ?? 0,   // Nullish coalescing
  };
}

crearUsuario("Andres", "a@test.com");         // OK
crearUsuario("Andres", "a@test.com", 30);     // OK

// Parametro con valor por defecto
function conectar(
  host: string = "localhost",
  puerto: number = 8080,
  tls: boolean = false,
): string {
  const protocolo = tls ? "https" : "http";
  return `${protocolo}://${host}:${puerto}`;
}

conectar();                          // http://localhost:8080
conectar("api.ejemplo.com");         // http://api.ejemplo.com:8080
conectar("api.ejemplo.com", 443, true); // https://api.ejemplo.com:443
```

### Parametros rest

```typescript
// Rest parameters: numero variable de argumentos
function sumarTodos(...numeros: number[]): number {
  return numeros.reduce((acc, n) => acc + n, 0);
}

console.log(sumarTodos(1, 2, 3, 4, 5)); // 15

// Rest con tipos mixtos
function loggearConPrefijo(prefijo: string, ...mensajes: string[]): void {
  for (const msg of mensajes) {
    console.log(`[${prefijo}] ${msg}`);
  }
}

loggearConPrefijo("INFO", "Servidor iniciado", "Puerto 8080");

// Spread operator (lo opuesto de rest)
const numeros = [1, 2, 3];
console.log(sumarTodos(...numeros));  // Equivalente a sumarTodos(1, 2, 3)
```

### Destructuracion de parametros

```typescript
interface OpcionesServidor {
  puerto: number;
  host: string;
  tls?: boolean;
}

// Destructuracion en la firma de la funcion
function iniciarServidor({ puerto, host, tls = false }: OpcionesServidor): void {
  console.log(`Servidor en ${host}:${puerto} (TLS: ${tls})`);
}

iniciarServidor({ puerto: 8080, host: "localhost" });
iniciarServidor({ puerto: 443, host: "api.ejemplo.com", tls: true });

// Con valores por defecto en la destructuracion
function configurar({
  puerto = 3000,
  host = "0.0.0.0",
  timeout = 30_000,
}: Partial<OpcionesServidor> = {}): OpcionesServidor {
  return { puerto, host, timeout };
}
```

---

## 4.2 Tipado de Funciones

### Type aliases para funciones

```typescript
// Type alias para firma de funcion
type Operacion = (a: number, b: number) => number;

const suma: Operacion = (a, b) => a + b;
const resta: Operacion = (a, b) => a - b;
const multiplicacion: Operacion = (a, b) => a * b;

// Type alias generico para callback
type Callback<T> = (error: Error | null, resultado?: T) => void;
type Predicado<T> = (item: T) => boolean;
type Transformador<T, U> = (item: T) => U;

// Uso con callbacks
function leerArchivo(ruta: string, cb: Callback<string>): void {
  try {
    const contenido = "contenido del archivo";
    cb(null, contenido);
  } catch (error) {
    cb(error as Error);
  }
}
```

### Interfaces para funciones (call signatures)

```typescript
// Interface con call signature
interface Comparador<T> {
  (a: T, b: T): number;
}

const porEdad: Comparador<{ edad: number }> = (a, b) => a.edad - b.edad;

// Interface hibrida: funcion con propiedades
interface Logger {
  (mensaje: string): void;
  nivel: "debug" | "info" | "error";
  prefix: string;
}

function crearLogger(prefix: string): Logger {
  const logger = ((mensaje: string) => {
    console.log(`[${prefix}] ${mensaje}`);
  }) as Logger;

  logger.nivel = "info";
  logger.prefix = prefix;
  return logger;
}

const log = crearLogger("APP");
log("Servidor iniciado");  // [APP] Servidor iniciado
console.log(log.nivel);    // info
```

---

## 4.3 Sobrecarga de Funciones

TypeScript permite declarar multiples firmas para una misma funcion:

```typescript
// Firmas de sobrecarga
function procesar(valor: string): string;
function procesar(valor: number): number;
function procesar(valor: string | number): string | number;

// Implementacion (debe ser compatible con todas las firmas)
function procesar(valor: string | number): string | number {
  if (typeof valor === "string") {
    return valor.toUpperCase();
  }
  return valor * 2;
}

const r1 = procesar("hola");   // string: "HOLA"
const r2 = procesar(21);        // number: 42
// const r3 = procesar(true);   // ❌ Error: boolean no sobrecargado

// Ejemplo practico: API con diferentes signatures
interface Usuario {
  id: number;
  nombre: string;
}

function buscarUsuario(id: number): Usuario | undefined;
function buscarUsuario(email: string): Usuario | undefined;
function buscarUsuario(criterio: number | string): Usuario | undefined {
  if (typeof criterio === "number") {
    return baseDeDatos.find((u) => u.id === criterio);
  }
  return baseDeDatos.find((u) => u.email === criterio);
}

// Sobrecarga con generics
function obtener<T>(clave: string): T | undefined;
function obtener<T>(clave: string, valorPorDefecto: T): T;
function obtener<T>(clave: string, valorPorDefecto?: T): T | undefined {
  const valor = cache.get(clave);
  if (valor !== undefined) return valor as T;
  return valorPorDefecto;
}

const puerto = obtener<number>("puerto", 8080); // number (no undefined)
const host = obtener<string>("host");            // string | undefined
```

---

## 4.4 Arrow Functions y this

### Arrow functions vs function declarations

```typescript
class Contador {
  private valor = 0;

  // function declaration: this depende de como se llame
  incrementarTradicional(): void {
    this.valor++;
  }

  // arrow function: this esta ligado lexicamente
  incrementarFlecha = (): void => {
    this.valor++;
  };

  programarIncremento(): void {
    // Con function declaration, this se pierde en el callback
    setTimeout(function () {
      // this.valor++; // ❌ Error: this no es Contador
    }, 1000);

    // Con arrow function, this se mantiene
    setTimeout(() => {
      this.valor++; // ✅ this es Contador
    }, 1000);

    // Alternativa: .bind()
    setTimeout(this.incrementarTradicional.bind(this), 1000);
  }
}
```

### Cuando usar cada una

```typescript
// Usa arrow function cuando:
// 1. Callbacks que necesitan this lexico
// 2. Funciones cortas con retorno implicito
// 3. Funciones anonimas en metodos funcionales

const numeros = [1, 2, 3, 4, 5];
const dobles = numeros.map((n) => n * 2);
const pares = numeros.filter((n) => n % 2 === 0);

// Usa function declaration cuando:
// 1. Metodos de clase que no necesitan this lexico
// 2. Funciones que necesitan hoisting
// 3. Funciones constructora (aunque hoy se usan clases)
```

---

## 4.5 Closures y Scope

```typescript
// Closure: funcion que captura variables de su scope lexico
function crearContador(inicial: number = 0) {
  let contador = inicial;

  return {
    incrementar: (): number => ++contador,
    decrementar: (): number => --contador,
    valor: (): number => contador,
    resetear: (): void => {
      contador = inicial;
    },
  };
}

const contador = crearContador(10);
console.log(contador.incrementar()); // 11
console.log(contador.incrementar()); // 12
console.log(contador.valor());       // 12
contador.resetear();
console.log(contador.valor());       // 10

// Patron: modulo revelador con tipos (Revealing Module)
function crearServicioCache<T>() {
  const cache = new Map<string, T>();

  return {
    guardar(clave: string, valor: T): void {
      cache.set(clave, valor);
    },
    obtener(clave: string): T | undefined {
      return cache.get(clave);
    },
    invalidar(clave: string): boolean {
      return cache.delete(clave);
    },
    tamano(): number {
      return cache.size;
    },
    limpiar(): void {
      cache.clear();
    },
  };
}

const cacheUsuarios = crearServicioCache<{ id: number; nombre: string }>();
cacheUsuarios.guardar("user-1", { id: 1, nombre: "Andres" });
console.log(cacheUsuarios.obtener("user-1")); // { id: 1, nombre: "Andres" }
```

### Cuidado con closures en bucles

```typescript
// PROBLEMA CLASICO (TypeScript lo detecta con noImplicitAny)
for (var i = 0; i < 5; i++) {
  setTimeout(() => console.log(i), 100); // 5, 5, 5, 5, 5 (con var)
}

// SOLUCION 1: let tiene scope de bloque
for (let i = 0; i < 5; i++) {
  setTimeout(() => console.log(i), 100); // 0, 1, 2, 3, 4 ✅
}

// SOLUCION 2: IIFE (Immediately Invoked Function Expression)
for (var i = 0; i < 5; i++) {
  ((j: number) => {
    setTimeout(() => console.log(j), 100);
  })(i);
}
```

---

## 4.6 Higher-Order Functions

Funciones que reciben o retornan otras funciones:

```typescript
// HOF: recibe funcion como parametro
function aplicarOperacion(
  numeros: number[],
  operacion: (n: number) => number,
): number[] {
  return numeros.map(operacion);
}

const resultado = aplicarOperacion([1, 2, 3], (n) => n * 2); // [2, 4, 6]

// HOF: retorna una funcion (factory)
function crearMultiplicador(factor: number): (n: number) => number {
  return (n: number): number => n * factor;
}

const doble = crearMultiplicador(2);
const triple = crearMultiplicador(3);
console.log(doble(5));   // 10
console.log(triple(5));  // 15

// HOF: composicion de funciones (pipe)
function pipe<T>(...fns: Array<(arg: T) => T>): (arg: T) => T {
  return (valor: T): T => fns.reduce((acc, fn) => fn(acc), valor);
}

const procesarTexto = pipe<string>(
  (s) => s.trim(),
  (s) => s.toLowerCase(),
  (s) => s.replace(/\s+/g, "_"),
);

console.log(procesarTexto("  Hola Mundo CRUEL  ")); // "hola_mundo_cruel"

// HOF: curry (transformar funcion multi-parametro en cadena)
function curry<T1, T2, R>(fn: (a: T1, b: T2) => R): (a: T1) => (b: T2) => R {
  return (a: T1) => (b: T2) => fn(a, b);
}

const sumaCurrificada = curry((a: number, b: number) => a + b);
const sumar5 = sumaCurrificada(5);
console.log(sumar5(3)); // 8
```

---

---

## 4.8 this, call, apply y bind

### Tipar this en funciones

```typescript
// this parameter (primer parametro falso, solo para tipos)
function saludar(this: { nombre: string }, saludo: string): string {
  return `${saludo}, ${this.nombre}!`;
}

const persona = { nombre: "Andres", saludar };
console.log(persona.saludar("Hola")); // "Hola, Andres!"

// Tipar this en callbacks
interface EventoClick {
  target: { id: string };
  timestamp: number;
}

function manejarClick(this: ElementoUI, evento: EventoClick): void {
  console.log(`Click en ${this.id}: ${evento.target.id}`);
}
```

### call, apply y bind

```typescript
function presentar(this: { nombre: string }, edad: number, ciudad: string): string {
  return `${this.nombre}, ${edad} años, de ${ciudad}`;
}

const usuario = { nombre: "Andres" };

// call: invoca con this y argumentos separados
console.log(presentar.call(usuario, 30, "Madrid"));

// apply: igual que call pero argumentos en array
console.log(presentar.apply(usuario, [30, "Madrid"]));

// bind: crea una nueva funcion con this fijo (no invoca)
const presentarAndres = presentar.bind(usuario);
console.log(presentarAndres(30, "Madrid"));

// bind parcial: fijar argumentos (currying)
const presentarEnMadrid = presentar.bind(usuario, 30, "Madrid");
console.log(presentarEnMadrid());
```

---

## 4.9 Memoization

```typescript
// Memoization: cachear resultados de funciones puras
function memoizar<T extends (...args: any[]) => any>(fn: T): T {
  const cache = new Map<string, ReturnType<T>>();

  return ((...args: Parameters<T>): ReturnType<T> => {
    const clave = JSON.stringify(args);

    if (cache.has(clave)) {
      return cache.get(clave)!;
    }

    const resultado = fn(...args);
    cache.set(clave, resultado);
    return resultado;
  }) as T;
}

// Funcion costosa (ej: fibonacci recursivo)
const fibonacci = memoizar((n: number): number => {
  if (n <= 1) return n;
  return fibonacci(n - 1) + fibonacci(n - 2);
});

console.time("fib(40)-primera");
console.log(fibonacci(40));       // Lento: calcula todo
console.timeEnd("fib(40)-primera");

console.time("fib(40)-segunda");
console.log(fibonacci(40));       // Instantaneo: cacheado
console.timeEnd("fib(40)-segunda");

// Memoizacion con opciones (TTL, maxSize)
function memoizarConTTL<T extends (...args: any[]) => any>(
  fn: T,
  ttlMs: number = 60_000,
): T {
  const cache = new Map<string, { valor: ReturnType<T>; expira: number }>();

  return ((...args: Parameters<T>): ReturnType<T> => {
    const clave = JSON.stringify(args);
    const entry = cache.get(clave);

    if (entry && Date.now() < entry.expira) {
      return entry.valor;
    }

    const resultado = fn(...args);
    cache.set(clave, { valor: resultado, expira: Date.now() + ttlMs });
    return resultado;
  }) as T;
}
```

---

## 4.10 Debounce y Throttle

```typescript
// Debounce: ejecuta solo cuando dejan de llegar llamadas
function debounce<T extends (...args: any[]) => void>(
  fn: T,
  esperaMs: number,
): (...args: Parameters<T>) => void {
  let timeoutId: NodeJS.Timeout;

  return (...args: Parameters<T>) => {
    clearTimeout(timeoutId);
    timeoutId = setTimeout(() => fn(...args), esperaMs);
  };
}

// Uso: buscar mientras el usuario escribe
const buscar = debounce(async (termino: string) => {
  console.log(`Buscando: ${termino}`);
  // fetch(`/api/buscar?q=${termino}`)...
}, 300);

buscar("n");    // No ejecuta (reinicia timer)
buscar("no");   // No ejecuta (reinicia timer)
buscar("nod");  // No ejecuta (reinicia timer)
buscar("node"); // Ejecuta despues de 300ms sin cambios

// Throttle: ejecuta como maximo una vez cada N ms
function throttle<T extends (...args: any[]) => void>(
  fn: T,
  intervaloMs: number,
): (...args: Parameters<T>) => void {
  let ultimaEjecucion = 0;

  return (...args: Parameters<T>) => {
    const ahora = Date.now();
    if (ahora - ultimaEjecucion >= intervaloMs) {
      ultimaEjecucion = ahora;
      fn(...args);
    }
  };
}

// Uso: manejar scroll o eventos frecuentes
const manejarScroll = throttle(() => {
  console.log("Scroll event procesado");
}, 100);

// Debounce con cancel
function debounceCancelable<T extends (...args: any[]) => void>(
  fn: T,
  esperaMs: number,
) {
  let timeoutId: NodeJS.Timeout;

  const ejecutar = (...args: Parameters<T>) => {
    clearTimeout(timeoutId);
    timeoutId = setTimeout(() => fn(...args), esperaMs);
  };

  ejecutar.cancel = () => clearTimeout(timeoutId);
  ejecutar.flush = (...args: Parameters<T>) => {
    clearTimeout(timeoutId);
    fn(...args);
  };

  return ejecutar;
}
```

---

## 4.11 Funciones Asincronas

```typescript
// Funcion async retorna Promise<T>
async function obtenerUsuario(id: number): Promise<Usuario> {
  const respuesta = await fetch(`/api/usuarios/${id}`);
  if (!respuesta.ok) {
    throw new Error(`Error HTTP: ${respuesta.status}`);
  }
  return respuesta.json() as Promise<Usuario>;
}

// Arrow function async
const buscarUsuarios = async (ids: number[]): Promise<Usuario[]> => {
  const promesas = ids.map((id) => obtenerUsuario(id));
  return Promise.all(promesas);
};

// Promise.allSettled: todas completan, exitosas o fallidas
async function buscarConFallos(ids: number[]): Promise<void> {
  const resultados = await Promise.allSettled(
    ids.map((id) => obtenerUsuario(id)),
  );

  for (const r of resultados) {
    if (r.status === "fulfilled") {
      console.log("Encontrado:", r.value.nombre);
    } else {
      console.error("Error:", r.reason);
    }
  }
}
```

Se explora la asincronia en profundidad en el Capitulo 7.

---

## Resumen del Capítulo

- TypeScript añade tipado de parametros y retorno a las funciones de JavaScript.
- Parametros opcionales (`?`), con valor por defecto y rest (`...args`).
- Type aliases e interfaces pueden describir firmas de funciones completas.
- La sobrecarga permite multiples firmas para una misma implementacion.
- Arrow functions ligan `this` lexicamente; ideales para callbacks.
- Los closures capturan el scope lexico; cuidado con bucles (`let` soluciona).
- HOFs: funciones que reciben/retornan funciones. Patrones: factory, pipe, curry.
- Las funciones async retornan siempre `Promise<T>`.
- Prefiere arrow functions para callbacks cortos y function declarations para metodos.

En el siguiente capitulo exploraremos las estructuras de datos en profundidad.
