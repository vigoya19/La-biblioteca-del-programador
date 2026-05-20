# Capítulo 9: Módulos, Paquetes y npm

El ecosistema de paquetes de Node.js es el mas grande del mundo con mas de 2 millones de paquetes. Entender CommonJS, ESM, y las herramientas de gestion es esencial para cualquier desarrollador Node.js.

---

## 9.1 CommonJS vs ESM

### CommonJS (CJS) - El sistema legacy

```typescript
// ─── modulo.ts (CommonJS) ───
// Exportar
module.exports = {
  saludar: (nombre: string): string => `Hola, ${nombre}`,
  PI: 3.14159,
};

// Exportaciones individuales
exports.sumar = (a: number, b: number): number => a + b;
module.exports.restar = (a: number, b: number): number => a - b;

// ─── consumo.ts ───
// Importar (con tipos en TypeScript)
import { saludar, PI } from "./modulo";
// o
const { saludar, PI } = require("./modulo");

// require dinamico
if (condicion) {
  const modulo = require("./modulo-condicional");
}
```

### ECMAScript Modules (ESM) - El estandar moderno

```json
// package.json
{
  "type": "module"  // Habilita ESM en todo el proyecto
}
```

```typescript
// ─── modulo.ts (ESM) ───
// Exportacion nombrada
export function saludar(nombre: string): string {
  return `Hola, ${nombre}`;
}

export const PI = 3.14159;

// Exportacion por defecto
export default class Usuario {
  constructor(public nombre: string) {}
}

// ─── consumo.ts ───
// Import nombrado
import { saludar, PI } from "./modulo.js"; // Nota: extension .js en imports!

// Import por defecto + nombrados
import Usuario, { saludar as hola } from "./modulo.js";

// Import namespace
import * as modulo from "./modulo.js";
console.log(modulo.saludar("Andres"));

// Import dinamico (async)
const modulo = await import("./modulo.js");
```

### Tabla comparativa

| Caracteristica | CommonJS | ESM |
|---------------|----------|-----|
| Sintaxis | `require()` / `module.exports` | `import` / `export` |
| Carga | Sincrona (bloqueante) | Asincrona |
| `this` en top-level | `module.exports` | `undefined` |
| `__dirname`, `__filename` | Disponible | Usar `import.meta.url` |
| `require` dinamico | `require()` en cualquier sitio | `import()` (async) |
| Tree shaking | No | Si (bundlers) |
| Top-level await | No | Si |
| Estricto por defecto | No | Si |

### ESM en TypeScript para Node.js

```json
// tsconfig.json para ESM
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "NodeNext",
    "moduleResolution": "NodeNext",
    "outDir": "./dist",
    "rootDir": "./src",
    "strict": true,
    "esModuleInterop": true
  }
}
```

```typescript
// src/index.ts
// Con module: "NodeNext", los imports relativos DEBEN incluir .js
import { saludar } from "./utils.js";  // .js, no .ts
import { readFile } from "node:fs/promises";  // node: prefijo

// __dirname en ESM
import { fileURLToPath } from "node:url";
import { dirname } from "node:path";

const __filename = fileURLToPath(import.meta.url);
const __dirname = dirname(__filename);
```

---

## 9.2 package.json en Profundidad

```json
{
  "name": "@mi-org/mi-api",
  "version": "1.0.0",
  "description": "API de usuarios con Node.js y TypeScript",
  "type": "module",
  "main": "./dist/index.js",
  "module": "./dist/index.js",
  "types": "./dist/index.d.ts",
  "exports": {
    ".": {
      "import": "./dist/index.js",
      "require": "./dist/index.cjs",
      "types": "./dist/index.d.ts"
    },
    "./utils": {
      "import": "./dist/utils/index.js",
      "types": "./dist/utils/index.d.ts"
    }
  },
  "files": [
    "dist",
    "README.md"
  ],
  "scripts": {
    "dev": "tsx watch src/index.ts",
    "build": "tsc",
    "start": "node dist/index.js",
    "test": "vitest run",
    "test:watch": "vitest",
    "lint": "biome lint src/",
    "format": "biome format --write src/",
    "typecheck": "tsc --noEmit",
    "clean": "rm -rf dist node_modules"
  },
  "dependencies": {
    "express": "^4.21.0",
    "zod": "^3.23.0",
    "prisma": "^5.19.0"
  },
  "devDependencies": {
    "typescript": "^5.6.0",
    "vitest": "^2.1.0",
    "@biomejs/biome": "^1.9.0",
    "@types/express": "^5.0.0",
    "@types/node": "^22.0.0",
    "tsx": "^4.19.0"
  },
  "engines": {
    "node": ">=22.0.0"
  },
  "packageManager": "pnpm@9.0.0"
}
```

---

## 9.3 npm, yarn y pnpm

```bash
# ─── npm ───
npm init -y                     # Inicializar
npm install                     # Instalar desde package-lock.json
npm install express zod         # Produccion
npm install -D vitest typescript # Desarrollo
npm update                      # Actualizar todo
npm uninstall express           # Remover
npm publish                     # Publicar en npm
npm audit                       # Auditar vulnerabilidades

# ─── pnpm (RECOMENDADO) ───
# Mas rapido, eficiente en disco, estricto (no ghost dependencies)
pnpm add express zod            # Produccion
pnpm add -D vitest typescript   # Desarrollo
pnpm update                     # Actualizar
pnpm remove express             # Remover

# ─── yarn (clasico) ───
yarn add express zod
yarn add -D vitest typescript
```

---

## 9.4 Versionado Semantico y Actualizacion

```typescript
// Versionado: MAJOR.MINOR.PATCH  (1.2.3)
// MAJOR: cambios incompatibles (breaking changes)
// MINOR: nueva funcionalidad compatible hacia atras
// PATCH: correcciones de bugs compatibles

// Rangos en package.json
{
  "dependencies": {
    "exacta": "4.21.0",           // Exactamente esta version
    "tilde": "~4.21.0",           // >=4.21.0 <4.22.0 (solo patch)
    "caret": "^4.21.0",           // >=4.21.0 <5.0.0 (minor y patch)
    "mayor": "4.x",               // >=4.0.0 <5.0.0
    "rango": ">=4.0.0 <5.0.0"    // Rango explicito
  }
}

// Verificar dependencias desactualizadas
// npx npm-check-updates
// npx ncu -u  // Actualizar package.json
```

---

## 9.5 Publicar Paquetes en npm

```bash
# 1. Preparar el paquete
npm login

# 2. Verificar que package.json tiene name, version, main/types
# 3. Compilar TypeScript
npm run build

# 4. Publicar (sin scope @org/)
npm publish

# Publicar con scope (@mi-org/mi-paquete)
npm publish --access public

# Publicar version especifica
npm version patch   # 1.0.0 -> 1.0.1
npm version minor   # 1.0.1 -> 1.1.0
npm version major   # 1.1.0 -> 2.0.0
npm publish

# Version pre-release
npm version prerelease --preid=beta  # -> 2.0.0-beta.0
```

### Buenas practicas al publicar

```json
{
  "files": ["dist"],
  "main": "./dist/index.js",
  "types": "./dist/index.d.ts",
  "sideEffects": false
}
```

```bash
# Verificar que se publicara lo correcto
npm pack --dry-run
```

---

## 9.6 Monorepos con npm Workspaces

```
mi-monorepo/
├── package.json          # Raiz: workspaces
├── packages/
│   ├── shared/           # @mi-app/shared
│   │   ├── src/
│   │   └── package.json
│   ├── api/              # @mi-app/api
│   │   ├── src/
│   │   └── package.json
│   └── web/              # @mi-app/web
│       ├── src/
│       └── package.json
└── tsconfig.base.json    # Configuracion base de TS
```

```json
// package.json (raiz)
{
  "private": true,
  "workspaces": ["packages/*"],
  "scripts": {
    "dev": "npm run dev --workspaces",
    "build": "npm run build --workspaces",
    "test": "npm run test --workspaces"
  }
}

// packages/shared/package.json
{
  "name": "@mi-app/shared",
  "version": "1.0.0",
  "main": "./dist/index.js",
  "types": "./dist/index.d.ts"
}
```

---

## 9.7 Path Aliases con TypeScript

```json
// tsconfig.json
{
  "compilerOptions": {
    "baseUrl": ".",
    "paths": {
      "@/*": ["./src/*"],
      "@shared/*": ["./packages/shared/src/*"],
      "@config/*": ["./src/config/*"]
    }
  }
}
```

```typescript
// Sin path aliases
import { validarEmail } from "../../../shared/utils/validacion";

// Con path aliases
import { validarEmail } from "@shared/utils/validacion";
import { config } from "@config/env";
```

### Resolver aliases en runtime (ESM)

```typescript
// Necesitas un package que resuelva los aliases en Node.js
// Opcion 1: tsx (lo maneja automaticamente)
// Opcion 2: tsc-alias (post-compilacion)
// Opcion 3: import maps (Node.js experimental)

// Para Jest/Vitest, configurar moduleNameMapper:
// vitest.config.ts
import { defineConfig } from "vitest/config";
import path from "node:path";

export default defineConfig({
  resolve: {
    alias: {
      "@": path.resolve(__dirname, "./src"),
    },
  },
});
```

---

## 9.8 Patrones de Organizacion de Proyectos

### Estructura recomendada (modular/hexagonal)

```
mi-api/
├── src/
│   ├── index.ts                 # Punto de entrada
│   ├── app.ts                   # Configuracion de Express/Fastify
│   ├── config/
│   │   ├── env.ts               # Variables de entorno tipadas
│   │   └── database.ts          # Config BD
│   ├── modules/
│   │   ├── usuarios/
│   │   │   ├── usuario.controller.ts
│   │   │   ├── usuario.service.ts
│   │   │   ├── usuario.repository.ts
│   │   │   ├── usuario.schema.ts   # Zod schemas
│   │   │   ├── usuario.routes.ts
│   │   │   └── usuario.test.ts
│   │   └── auth/
│   │       ├── auth.controller.ts
│   │       ├── auth.service.ts
│   │       └── auth.test.ts
│   └── shared/
│       ├── errors.ts            # Jerarquia de errores
│       ├── logger.ts            # Configuracion winston/pino
│       ├── middleware.ts        # Middlewares compartidos
│       └── types.ts             # Tipos compartidos
├── tests/
│   └── integration/
├── prisma/                      # Schema y migraciones
│   └── schema.prisma
├── .env.example
├── tsconfig.json
├── vitest.config.ts
├── biome.json
└── package.json
```

---

## 9.9 Dual Publishing (CJS + ESM)

Publicar un paquete que funcione como CommonJS y ESM simultaneamente:

```json
// package.json
{
  "name": "mi-libreria",
  "exports": {
    ".": {
      "import": "./dist/index.js",     // ESM: import { x } from "mi-libreria"
      "require": "./dist/index.cjs",   // CJS: const { x } = require("mi-libreria")
      "types": "./dist/index.d.ts"
    }
  },
  "main": "./dist/index.cjs",          // Fallback CJS
  "module": "./dist/index.js",         // Fallback ESM (bundlers)
  "types": "./dist/index.d.ts",
  "files": ["dist"],
  "type": "module"                     // .ts se compilan a .js (ESM)
}
```

```typescript
// tsconfig.json para dual publish
{
  "compilerOptions": {
    "outDir": "./dist",
    "module": "NodeNext",
    "declaration": true
    // No usar "type": "module" en el tsconfig del paquete
  }
}
```

```bash
# Compilar a ESM (.js) y CJS (.cjs) por separado
# Opcion 1: dos tsconfigs
tsc -p tsconfig.esm.json
tsc -p tsconfig.cjs.json

# Opcion 2: usar un bundler (tsup, unbuild)
npx tsup src/index.ts --format esm,cjs --dts
```

### Extensiones: .mjs, .cjs, .mts, .cts

| Extension | Significado | Usado en |
|-----------|-------------|----------|
| `.mjs` | ESM explicito (ignora "type" en package.json) | Node.js |
| `.cjs` | CJS explicito | Node.js |
| `.mts` | TypeScript fuente para ESM | TypeScript |
| `.cts` | TypeScript fuente para CJS | TypeScript |

---

## 9.10 peerDependencies

```json
{
  "name": "mi-plugin-express",
  "peerDependencies": {
    "express": "^4.0.0 || ^5.0.0"
  },
  "peerDependenciesMeta": {
    "express": { "optional": false }
  }
}
```

```typescript
// peerDependencies: "mi paquete funciona CON express, pero no lo instalo yo"
// El proyecto consumidor es responsable de instalar express

// Ejemplos clasicos:
// - Plugins de frameworks (express middleware, NestJS modules)
// - Adaptadores de BD (@prisma/client como peer)
// - Librerias de React/Vue/Next.js

// npm 7+ auto-instala peerDependencies
// pnpm las requiere explicitas (estricto, mejor)
```

---

## 9.11 import.meta (Node.js 22+)

```typescript
// Estos requieren Node.js 22+ y --experimental-import-meta-resolve (o 23+ sin flag)

// import.meta.dirname: reemplaza __dirname
console.log(import.meta.dirname);

// import.meta.filename: reemplaza __filename
console.log(import.meta.filename);

// import.meta.resolve: resolver rutas de modulos
const ruta = import.meta.resolve("@/utils/logger");
console.log(ruta); // file:///Users/.../src/utils/logger.ts

// Antes (Node <22) necesitabas esto:
import { fileURLToPath } from "node:url";
import { dirname } from "node:path";
const __filename = fileURLToPath(import.meta.url);
const __dirname = dirname(__filename);
```

---

## 9.12 Turborepo (Monorepos Avanzados)

```bash
# Inicializar monorepo con Turborepo
npx create-turbo@latest

# Estructura
mi-monorepo/
├── package.json          # workspaces: ["apps/*", "packages/*"]
├── turbo.json
├── apps/
│   ├── api/              # @mi-app/api (NestJS backend)
│   └── web/              # @mi-app/web (Next.js frontend)
└── packages/
    ├── shared/           # @mi-app/shared (tipos, utils)
    ├── ui/               # @mi-app/ui (componentes React)
    └── config/           # @mi-app/config (tsconfig, eslint, biome)
```

```json
// turbo.json
{
  "$schema": "https://turbo.build/schema.json",
  "globalDependencies": ["**/.env.*local"],
  "tasks": {
    "build": {
      "dependsOn": ["^build"],
      "outputs": ["dist/**", ".next/**"]
    },
    "dev": {
      "cache": false,
      "persistent": true
    },
    "test": {
      "dependsOn": ["build"],
      "inputs": ["src/**/*.ts", "tests/**/*.ts"]
    },
    "lint": {},
    "typecheck": {
      "dependsOn": ["^build"]
    }
  }
}
```

```bash
# Turborepo cachea resultados y paraleliza
turbo build      # Solo re-compila lo que cambio
turbo dev        # Ejecuta dev en todos los paquetes
turbo test       # Solo re-testea lo afectado por cambios
turbo lint --filter=@mi-app/api  # Solo un paquete
```

---

## Resumen del Capítulo

- ESM (`import`/`export`) es el estandar moderno. CommonJS es legacy pero aun ubicuo.
- Usa `"type": "module"` en package.json para habilitar ESM en Node.js.
- TypeScript con `module: "NodeNext"` requiere extension `.js` en imports relativos.
- `pnpm` es el gestor recomendado: mas rapido, eficiente, y estricto.
- El versionado semantico usa MAJOR.MINOR.PATCH. El caret `^` es el rango por defecto.
- `exports` en package.json define la API publica y permite multiples entry points.
- npm workspaces facilitan monorepos sin herramientas externas.
- Path aliases (`@/`) simplifican imports profundos. Configurar en tsconfig y testing.

En el siguiente capitulo exploraremos el testing en Node.js y TypeScript.

---

← [Capítulo anterior](08-manejo-de-errores.md) | [Inicio](README.md) | [Capítulo siguiente →](10-testing.md)
