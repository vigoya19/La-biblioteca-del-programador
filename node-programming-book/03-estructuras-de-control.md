# Capítulo 3: Estructuras de Control

Las estructuras de control en TypeScript son identicas a JavaScript, pero con el beneficio del type narrowing automatico: TypeScript refina los tipos dentro de condicionales y bucles, haciendo el codigo mas seguro.

---

## 3.1 Condicionales

### if / else con type narrowing

```typescript
function procesar(valor: string | number | null): string {
  if (valor === null) {
    return "Valor nulo";
  }

  // TypeScript sabe que aqui valor es string | number (no null)
  if (typeof valor === "string") {
    return valor.toUpperCase();  // string
  }

  return valor.toFixed(2);       // number
}
```

### if con inicializacion (similar al patron Go)

```typescript
// TypeScript no tiene "if con declaracion" como Go, pero se simula asi:
const resultado = await fetch("/api/usuarios").catch(() => null);
if (resultado && resultado.ok) {
  const datos = await resultado.json();
  console.log(datos);
}

// O con bloque inmediato para scope limitado
{
  const usuario = await buscarUsuario(1);
  if (usuario) {
    console.log(usuario.nombre);
  }
}
// usuario no existe aqui (scope de bloque)
```

### Operador ternario con tipos

```typescript
const edad = 20;
const categoria: "menor" | "adulto" = edad >= 18 ? "adulto" : "menor";

// Con tipos complejos
interface RespuestaExitosa {
  ok: true;
  datos: string;
}

interface RespuestaFallida {
  ok: false;
  error: string;
}

type Respuesta = RespuestaExitosa | RespuestaFallida;

const respuesta: Respuesta = { ok: true, datos: "exito" };
const mensaje = respuesta.ok ? respuesta.datos : respuesta.error;
// TypeScript infiere: mensaje es string (union de ambos branches)
```

---

## 3.2 switch

### Switch basico

```typescript
type MetodoHTTP = "GET" | "POST" | "PUT" | "DELETE";

function manejarMetodo(metodo: MetodoHTTP): string {
  switch (metodo) {
    case "GET":
      return "Leer recurso";
    case "POST":
      return "Crear recurso";
    case "PUT":
      return "Actualizar recurso";
    case "DELETE":
      return "Eliminar recurso";
    default:
      // TypeScript verifica que cubrimos todos los casos
      // Si falta algun caso, da error con --noImplicitReturns
      const _exhaustivo: never = metodo;
      throw new Error(`Metodo no manejado: ${_exhaustivo}`);
  }
}
```

### Exhaustiveness checking con never

```typescript
type Estado =
  | { tipo: "cargando" }
  | { tipo: "exito"; datos: unknown }
  | { tipo: "error"; mensaje: string };

function renderizar(estado: Estado): string {
  switch (estado.tipo) {
    case "cargando":
      return "Cargando...";
    case "exito":
      return `Datos: ${JSON.stringify(estado.datos)}`;
    case "error":
      return `Error: ${estado.mensaje}`;
    default:
      // Si agregas un nuevo estado en el futuro,
      // TypeScript marcara error aqui: "estado is not never"
      const _exhaustivo: never = estado;
      return _exhaustivo;
  }
}
```

### Switch sin break (fall-through intencional)

```typescript
enum NivelLog {
  DEBUG = 0,
  INFO = 1,
  WARN = 2,
  ERROR = 3,
}

function debeLoguear(nivelConfig: NivelLog, nivelMensaje: NivelLog): boolean {
  // Sin break, todos los cases bajo el match se ejecutan
  switch (nivelMensaje) {
    case NivelLog.ERROR:
    case NivelLog.WARN:
      return nivelConfig <= nivelMensaje;
    case NivelLog.INFO:
    case NivelLog.DEBUG:
      return nivelConfig <= nivelMensaje;
    default:
      return false;
  }
}
```

---

## 3.3 Bucles

### for clasico

```typescript
// Tipado completo del indice
for (let i: number = 0; i < 10; i++) {
  console.log(`Iteracion ${i}`);
}

// Array con tipos
const frutas: string[] = ["manzana", "banana", "cereza"];
for (let i = 0; i < frutas.length; i++) {
  console.log(`${i}: ${frutas[i]}`);
}
```

### for...of (iterar valores) - RECOMENDADO

```typescript
const numeros: number[] = [1, 2, 3, 4, 5];

for (const n of numeros) {
  console.log(n * 2);
}

// Con destructuracion
const usuarios = [
  { id: 1, nombre: "Andres" },
  { id: 2, nombre: "Maria" },
];

for (const { id, nombre } of usuarios) {
  console.log(`Usuario ${id}: ${nombre}`);
}

// for...of con Map y Set
const edades = new Map<string, number>([
  ["Andres", 30],
  ["Maria", 25],
]);

for (const [nombre, edad] of edades) {
  console.log(`${nombre}: ${edad}`);
}
```

### for...in (iterar claves de objeto)

```typescript
interface Configuracion {
  puerto: number;
  host: string;
  debug: boolean;
}

const config: Configuracion = {
  puerto: 8080,
  host: "localhost",
  debug: false,
};

// for...in itera las claves (como string)
for (const clave in config) {
  // Type narrowing: clave es string, necesitamos type assertion
  const valor = config[clave as keyof Configuracion];
  console.log(`${clave}: ${valor}`);
}

// MEJOR: Object.entries con tipos (mas seguro)
for (const [clave, valor] of Object.entries(config)) {
  console.log(`${clave}: ${valor}`);
  // clave es string, valor es number | string | boolean
}
```

### while y do...while

```typescript
// while: evalua antes de cada iteracion
let intentos = 0;
while (intentos < 3) {
  console.log(`Intento ${intentos + 1}`);
  intentos++;
}

// do...while: ejecuta al menos una vez antes de evaluar
let contador = 0;
do {
  console.log(`Contador: ${contador}`);
  contador++;
} while (contador < 5);

// Patron: retry con backoff exponencial
async function conReintentos<T>(
  fn: () => Promise<T>,
  maxIntentos: number,
  esperaMs: number,
): Promise<T> {
  let intento = 0;
  while (intento < maxIntentos) {
    try {
      return await fn();
    } catch (error) {
      intento++;
      if (intento >= maxIntentos) throw error;
      const espera = esperaMs * Math.pow(2, intento);
      console.log(`Intento ${intento} fallido, reintentando en ${espera}ms`);
      await new Promise((resolve) => setTimeout(resolve, espera));
    }
  }
  throw new Error("Inalcanzable");
}
```

---

## 3.4 break, continue y Labels

```typescript
// break: sale del bucle
for (let i = 0; i < 10; i++) {
  if (i === 5) break;
  console.log(i); // 0, 1, 2, 3, 4
}

// continue: salta a la siguiente iteracion
for (let i = 0; i < 10; i++) {
  if (i % 2 === 0) continue;
  console.log(i); // 1, 3, 5, 7, 9
}

// Labels: romper bucles anidados
externo: for (let i = 0; i < 3; i++) {
  for (let j = 0; j < 3; j++) {
    if (i === 1 && j === 1) {
      break externo; // Sale de ambos bucles
    }
    console.log(`(${i}, ${j})`);
  }
}
// Imprime: (0,0) (0,1) (0,2) (1,0)
```

---

## 3.5 Patrones de Iteracion Funcional

En TypeScript moderno, muchas iteraciones se reemplazan por metodos funcionales:

```typescript
const usuarios = [
  { id: 1, nombre: "Andres", edad: 30, activo: true },
  { id: 2, nombre: "Maria", edad: 25, activo: false },
  { id: 3, nombre: "Carlos", edad: 35, activo: true },
];

// En vez de for...of con if, usa filter + map
const nombresActivos = usuarios
  .filter((u) => u.activo)
  .map((u) => u.nombre);
// ["Andres", "Carlos"]

// reduce: acumular valores
const edadTotal = usuarios.reduce((suma, u) => suma + u.edad, 0); // 90
const promedioEdad = edadTotal / usuarios.length; // 30

// some / every: condiciones sobre arrays
const hayInactivos = usuarios.some((u) => !u.activo);  // true
const todosAdultos = usuarios.every((u) => u.edad >= 18); // true

// find / findIndex: buscar elementos
const andres = usuarios.find((u) => u.nombre === "Andres");
const indiceMaria = usuarios.findIndex((u) => u.nombre === "Maria");

// flatMap: map + flat (aplanar arrays anidados)
const etiquetas = usuarios.flatMap((u) => [u.nombre, `${u.edad} años`]);
// ["Andres", "30 años", "Maria", "25 años", "Carlos", "35 años"]
```

### Iteradores personalizados

```typescript
class Rango implements Iterable<number> {
  constructor(
    private inicio: number,
    private fin: number,
  ) {}

  *[Symbol.iterator](): Iterator<number> {
    for (let i = this.inicio; i <= this.fin; i++) {
      yield i;
    }
  }
}

const rango = new Rango(1, 5);
for (const n of rango) {
  console.log(n); // 1, 2, 3, 4, 5
}

// Convertir a array con spread
const numeros = [...new Rango(1, 10)];
console.log(numeros); // [1, 2, ..., 10]

// Generadores con tipos
function* fibonacci(n: number): Generator<number> {
  let a = 0, b = 1;
  for (let i = 0; i < n; i++) {
    yield a;
    [a, b] = [b, a + b];
  }
}

const secuencia = [...fibonacci(10)];
console.log(secuencia); // [0, 1, 1, 2, 3, 5, 8, 13, 21, 34]
```

---

## Resumen del Capítulo

- TypeScript refina tipos automaticamente dentro de condicionales (type narrowing).
- El operador ternario mantiene la seguridad de tipos cuando ambos branches son compatibles.
- `switch` con discriminated unions y exhaustiveness checking (`never`) garantiza cubrir todos los casos.
- `for...of` es la forma recomendada de iterar arrays y otros iterables.
- `for...in` itera claves; mejor usar `Object.entries()` para type safety.
- Los metodos funcionales (`map`, `filter`, `reduce`) reemplazan la mayoria de los bucles.
- Generadores y `Symbol.iterator` permiten crear iterables personalizados.
- `break` con labels permite salir de bucles anidados.

En el siguiente capitulo exploraremos las funciones en profundidad.

---

← [Capítulo anterior](02-sintaxis-y-tipos.md) | [Inicio](README.md) | [Capítulo siguiente →](04-funciones.md)
