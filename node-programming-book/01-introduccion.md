# Capítulo 1: Introducción a Node.js y TypeScript

Node.js revoluciono el desarrollo web al llevar JavaScript al servidor. TypeScript añadio seguridad de tipos y herramientas de desarrollo que transformaron la experiencia de programar en JavaScript a escala empresarial.

---

## 1.1 Historia y Filosofia de Node.js

Node.js fue creado en 2009 por **Ryan Dahl**. Su idea era simple pero revolucionaria: usar el motor V8 de Google Chrome fuera del navegador para ejecutar JavaScript en el servidor, combinado con un modelo de I/O no bloqueante y orientado a eventos.

### ¿Por que se creo Node.js?

Los servidores web tradicionales (Apache, Tomcat) usaban un modelo de un hilo por conexion. Esto funcionaba para pocos usuarios pero colapsaba bajo cargas altas de conexiones concurrentes (el problema C10K: 10,000 conexiones simultaneas).

Node.js resolvio esto con:

- **Event Loop**: un solo hilo principal que maneja miles de conexiones concurrentes mediante un bucle de eventos.
- **I/O no bloqueante**: operaciones de red, archivos y bases de datos nunca bloquean el hilo principal.
- **libuv**: libreria en C que proporciona el event loop y operaciones asincronas multiplataforma.

```typescript
// Modelo bloqueante tradicional (pseudo-codigo)
// while (true) {
//   const conn = acceptConnection()  // Bloquea hasta que llega una conexion
//   const data = readFromSocket(conn) // Bloquea hasta que llegan datos
//   const result = queryDatabase(data) // Bloquea hasta que responde la BD
//   writeToSocket(conn, result)       // Bloquea hasta que se envia
// }
// Solo UNA conexion a la vez. Las demas esperan.

// Modelo Node.js (no bloqueante, orientado a eventos)
import { createServer } from "node:http";

const server = createServer((req, res) => {
  // Este callback se ejecuta cuando llega una peticion
  // Miles de peticiones pueden estar en vuelo simultaneamente
  res.writeHead(200, { "Content-Type": "application/json" });
  res.end(JSON.stringify({ mensaje: "Hola, mundo!" }));
});

server.listen(3000, () => {
  console.log("Servidor escuchando en http://localhost:3000");
});
```

### Filosofia de Node.js

1. **JavaScript everywhere**: mismo lenguaje en frontend y backend.
2. **Event-driven**: todo gira alrededor de eventos asincronos.
3. **Non-blocking I/O**: nunca bloquear el event loop.
4. **npm ecosystem**: el registro de paquetes mas grande del mundo.
5. **Small core, large ecosystem**: nucleo pequeño, funcionalidad via modulos.

### Node.js en la industria

- **Netflix**: migro a Node.js reduciendo tiempo de inicio de 40min a <1min.
- **PayPal**: Node.js maneja transacciones para 400M+ usuarios.
- **LinkedIn**: Node.js en su backend movil, 20x mas rapido que Ruby.
- **Uber, Trello, eBay, NASA, Walmart**: servicios criticos en Node.js.

---

## 1.2 ¿Por que TypeScript?

TypeScript fue creado por **Anders Hejlsberg** (creador de C# y Turbo Pascal) en Microsoft en 2012. Es un superset de JavaScript que añade tipado estatico opcional.

### Problemas que resuelve TypeScript

```typescript
// JAVASCRIPT: errores que solo ves en runtime
function saludar(usuario) {
  return "Hola, " + usuario.nombre;
}
saludar(null); // TypeError: Cannot read properties of null

function sumar(a, b) {
  return a + b;
}
sumar(1, "2"); // "12" (concatenacion inesperada)
sumar(1);       // NaN (parametro faltante)

// TYPESCRIPT: errores en tiempo de compilacion
interface Usuario {
  nombre: string;
  edad: number;
}

function saludar(usuario: Usuario): string {
  return `Hola, ${usuario.nombre}`;
}
// saludar(null);        // ❌ Error de compilacion
// sumar(1, "2");        // ❌ Error de compilacion
// sumar(1);             // ❌ Error de compilacion
```

### Beneficios clave

| Beneficio | Sin TypeScript | Con TypeScript |
|-----------|---------------|----------------|
| **Deteccion de errores** | En runtime (produccion) | En compilacion (editor) |
| **Autocompletado** | Limitado, adivinatorio | Preciso, contextual |
| **Refactorizacion** | Peligrosa (buscar/reemplazar) | Segura (renombrar con confianza) |
| **Documentacion** | Comentarios (se desactualizan) | Tipos (el compilador los verifica) |
| **Navegacion** | Grep / buscar texto | Go-to-definition, find references |

---

## 1.3 Instalacion y Configuracion del Entorno

### Instalar Node.js

```bash
# ─── fnm (Fast Node Manager) - RECOMENDADO ───
# macOS/Linux
curl -fsSL https://fnm.vercel.app/install | bash

# Instalar y usar version LTS
fnm install --lts
fnm use lts-latest
fnm default lts-latest

# Alternativa: volta (rust, cross-platform)
curl https://get.volta.sh | bash
volta install node@lts
volta install node@22

# Alternativa: nvm (Node Version Manager)
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.0/install.sh | bash
nvm install --lts
nvm alias default lts/*

# macOS con Homebrew
brew install node

# Linux (Ubuntu/Debian)
curl -fsSL https://deb.nodesource.com/setup_22.x | sudo -E bash -
sudo apt-get install -y nodejs
```

### `.nvmrc` y `.node-version`

```bash
# En la raiz de tu proyecto, especificar version de Node
echo "22" > .nvmrc         # nvm usa esto
echo "22" > .node-version   # fnm usa esto (y nvm como fallback)

# Al entrar al directorio, nvm/fnm cambian automaticamente
# (requiere hooks en el shell)
```

### corepack (gestor de package managers)

> **📖 Para principiantes**: `npm` (que viene con Node.js) es todo lo que necesitas para empezar. Es el "gestor de paquetes" que te permite instalar librerías: escribes `npm install express` y se descarga lista para usar. Las herramientas de abajo (corepack, pnpm, yarn) son **alternativas avanzadas** que ofrecen mayor velocidad o mejor manejo de dependencias. Si estás empezando, **usa `npm` y salta esta sección sin preocupaciones**.

```bash
# corepack viene con Node.js 16+. Gestiona pnpm/yarn automaticamente
corepack enable

# Al ejecutar pnpm/yarn, corepack usa la version del package.json
# package.json: "packageManager": "pnpm@9.0.0"
pnpm --version  # Usa automaticamente 9.0.0
```

### Verificar la instalacion

```bash
node --version    # v22.11.0
npm --version     # 10.9.0
corepack --version
```

### Debugging con VS Code

> **📖 Para principiantes**: El "debugging" (depuración) es el proceso de encontrar y corregir errores en tu código. VS Code te permite ejecutar tu programa paso a paso, pausar la ejecución en cualquier línea, e inspeccionar el valor de las variables en ese momento. La configuración de abajo (archivo `launch.json`) le dice a VS Code cómo ejecutar tu programa en modo debug. No necesitas memorizarla — VS Code puede generarla automáticamente desde el menú "Run > Add Configuration".

```json
// .vscode/launch.json
{
  "version": "0.2.0",
  "configurations": [
    {
      "type": "node",
      "request": "launch",
      "name": "Debug con tsx",
      "runtimeExecutable": "tsx",
      "args": ["src/index.ts"],
      "console": "integratedTerminal",
      "skipFiles": ["<node_internals>/**"]
    },
    {
      "type": "node",
      "request": "attach",
      "name": "Atachar a proceso",
      "port": 9229,
      "restart": true
    }
  ]
}
```

```bash
# Atachar debugger a un proceso en ejecucion
node --inspect dist/index.js            # Chrome DevTools + VS Code
node --inspect-brk dist/index.js        # Pausa al inicio (primer statement)
# Abrir chrome://inspect en Chrome

# Hot reload con tsx watch
npx tsx watch src/index.ts
nodemon --exec "tsx" src/index.ts
```

### Instalar TypeScript

```bash
# Global (para usar tsc desde cualquier terminal)
npm install -g typescript

# Verificar
tsc --version     # Version 5.x.x
```

### Tu primer proyecto Node.js + TypeScript

```bash
# Crear directorio del proyecto
mkdir mi-proyecto
cd mi-proyecto

# Inicializar npm
npm init -y

# Instalar TypeScript como dependencia de desarrollo
npm install -D typescript @types/node

# Crear tsconfig.json
npx tsc --init
```

### tsconfig.json optimo para Node.js

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "NodeNext",
    "moduleResolution": "NodeNext",
    "outDir": "./dist",
    "rootDir": "./src",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "resolveJsonModule": true,
    "declaration": true,
    "declarationMap": true,
    "sourceMap": true
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "dist"]
}
```

### Ejecutar TypeScript en Node.js

```bash
# Opcion 1: Compilar y ejecutar (produccion)
npx tsc
node dist/index.js

# Opcion 2: tsx - ejecucion directa (desarrollo) - RECOMENDADO
npm install -D tsx
npx tsx src/index.ts

# Opcion 3: ts-node (legacy, menos popular hoy)
npm install -D ts-node
npx ts-node src/index.ts

# Opcion 4: Node.js 22+ con type stripping experimental
node --experimental-strip-types src/index.ts
```

---

## 1.4 Hola Mundo y Estructura Basica

### Tu primer programa

```typescript
// src/index.ts
const mensaje: string = "Hola, mundo desde Node.js + TypeScript!";
console.log(mensaje);

// Con tipado completo
function saludar(nombre: string): string {
  return `Hola, ${nombre}!`;
}

console.log(saludar("Andres"));
console.log(`Node.js ${process.version}`);
```

Ejecutalo:

```bash
npx tsx src/index.ts
# Hola, mundo desde Node.js + TypeScript!
# Hola, Andres!
# Node.js v22.11.0
```

### Anatomia de un proyecto Node.js + TypeScript

```
mi-proyecto/
├── src/                    # Codigo fuente TypeScript
│   ├── index.ts            # Punto de entrada
│   ├── config/
│   │   └── env.ts          # Configuracion de entorno
│   ├── modules/
│   │   └── usuario/
│   │       ├── usuario.controller.ts
│   │       ├── usuario.service.ts
│   │       ├── usuario.repository.ts
│   │       └── usuario.types.ts
│   └── shared/
│       ├── errors.ts
│       └── logger.ts
├── dist/                   # Codigo compilado (generado por tsc)
├── tests/                  # Tests
├── node_modules/           # Dependencias (generado por npm)
├── .env                    # Variables de entorno (NO committear)
├── .env.example            # Template de variables de entorno
├── .gitignore              # Archivos ignorados por git
├── .eslintrc.json          # Configuracion de ESLint
├── tsconfig.json           # Configuracion de TypeScript
├── package.json            # Metadata y dependencias
└── README.md               # Documentacion
```

---

## 1.5 Herramientas del Ecosistema

### npm (Node Package Manager)

```bash
# Inicializar proyecto
npm init -y

# Instalar dependencias
npm install express zod          # Produccion
npm install -D typescript vitest # Desarrollo

# Instalar version especifica
npm install express@4.21.0

# Actualizar
npm update
npm update express

# Desinstalar
npm uninstall express

# Ejecutar scripts definidos en package.json
npm run dev
npm run build
npm run test

# npx: ejecutar paquetes sin instalarlos globalmente
npx tsx src/index.ts
npx create-next-app@latest
npx npm-check-updates
```

### Gestores de paquetes alternativos

```bash
# pnpm: mas rapido, eficiente en disco (RECOMENDADO)
npm install -g pnpm
pnpm install
pnpm add express

# yarn: clasico, estable
npm install -g yarn
yarn add express

# bun: runtime + gestor todo-en-uno (mas rapido)
bun install
bun add express
```

### Scripts en package.json

```json
{
  "name": "mi-api",
  "version": "1.0.0",
  "type": "module",
  "scripts": {
    "dev": "tsx watch src/index.ts",
    "build": "tsc",
    "start": "node dist/index.js",
    "test": "vitest run",
    "test:watch": "vitest",
    "test:coverage": "vitest run --coverage",
    "lint": "biome lint src/",
    "format": "biome format --write src/",
    "typecheck": "tsc --noEmit"
  }
}
```

### Biome: formateo y linting moderno

```bash
# Instalar
npm install -D @biomejs/biome

# Inicializar configuracion
npx biome init

# biome.json
{
  "$schema": "https://biomejs.dev/schemas/1.9.4/schema.json",
  "organizeImports": {
    "enabled": true
  },
  "formatter": {
    "indentStyle": "space",
    "indentWidth": 2,
    "lineWidth": 100
  },
  "linter": {
    "enabled": true,
    "rules": {
      "recommended": true,
      "correctness": {
        "noUnusedVariables": "error"
      },
      "style": {
        "useConst": "error",
        "useTemplate": "error"
      }
    }
  },
  "javascript": {
    "formatter": {
      "semicolons": "always",
      "quoteStyle": "double",
      "trailingCommas": "all"
    }
  }
}
```

---

## 1.6 El Sistema de Tipos de TypeScript en 5 Minutos

```typescript
// Tipos basicos con anotacion explicita
const nombre: string = "Andres";
const edad: number = 30;
const activo: boolean = true;
const hobbies: string[] = ["programar", "leer"];
const cuenta: null = null;
const sinDefinir: undefined = undefined;

// Type inference: TypeScript deduce el tipo automaticamente
const apellido = "Garcia";       // string
const puntaje = 100;              // number
const items = ["a", "b", "c"];  // string[]

// Objetos con interfaces
interface Usuario {
  id: number;
  nombre: string;
  email: string;
  edad?: number; // Opcional
}

const usuario: Usuario = {
  id: 1,
  nombre: "Andres",
  email: "andres@ejemplo.com",
};

// Funciones con tipos
function sumar(a: number, b: number): number {
  return a + b;
}

const multiplicar = (a: number, b: number): number => a * b;

// Unions: una variable puede ser de varios tipos
type Estado = "activo" | "inactivo" | "pendiente";
let estado: Estado = "activo";

// Generics: tipos que aceptan otros tipos.
// Los generics son como una "caja sorpresa reutilizable": la funcion no sabe
// de antemano si trabajara con numeros, textos o lo que sea. La <T> es un
// comodin que dice "T puede ser cualquier tipo" y TypeScript lo infiere
// automaticamente segun lo que le pases.
function primerElemento<T>(arr: T[]): T | undefined {
  return arr[0];
}

const primero = primerElemento([1, 2, 3]); // number | undefined
const primeraLetra = primerElemento(["a", "b"]); // string | undefined
```

---

## Resumen del Capítulo

- Node.js ejecuta JavaScript en el servidor con un modelo de I/O no bloqueante basado en eventos.
- TypeScript añade tipado estatico, detectando errores en tiempo de compilacion en vez de en produccion.
- Las herramientas clave son: `node`, `npm`/`pnpm`, `tsc`, `tsx`, `biome`.
- Un proyecto Node.js tipico tiene `src/` (codigo), `dist/` (compilado), `tests/` y `package.json`.
- El `tsconfig.json` configura como TypeScript compila tu codigo. `strict: true` es esencial.
- Los scripts en `package.json` automatizan tareas comunes (dev, build, test, lint).
- El ecosistema npm es el mas grande del mundo con 2M+ paquetes disponibles.

En el siguiente capitulo exploraremos la sintaxis y los tipos de TypeScript en detalle.

---

[Inicio](README.md) | [Capítulo siguiente →](02-sintaxis-y-tipos.md)
