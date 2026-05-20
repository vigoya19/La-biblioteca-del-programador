# Node.js + TypeScript: Guía Completa de Programación

## Índice General

1. [Capítulo 1: Introducción a Node.js y TypeScript](01-introduccion.md)
   - Historia y filosofía de Node.js
   - ¿Por qué TypeScript?
   - Configuración del entorno de desarrollo
   - Hola Mundo y estructura básica
   - Herramientas del ecosistema (npm, npx, tsx, ts-node, Biome)

2. [Capítulo 2: Sintaxis y Tipos en TypeScript](02-sintaxis-y-tipos.md)
   - Variables (let, const, var) y su alcance
   - Tipos básicos (number, string, boolean, null, undefined, symbol, bigint)
   - Tipos literales y union types
   - Type assertions y type narrowing
   - Operadores
   - Enums y const assertions

3. [Capítulo 3: Estructuras de Control](03-estructuras-de-control.md)
   - Condicionales (if, else, switch, operador ternario)
   - Bucles (for, while, do-while, for...of, for...in)
   - break, continue, labels
   - Pattern matching con switch (avanzado)

4. [Capítulo 4: Funciones](04-funciones.md)
   - Declaración y parámetros
   - Tipado de funciones
   - Parámetros opcionales, por defecto y rest
   - Funciones flecha (arrow functions)
   - Closures y scope
   - Funciones como valores y tipos
   - Sobrecarga de funciones
   - Higher-order functions

5. [Capítulo 5: Estructuras de Datos](05-estructuras-de-datos.md)
   - Arrays y métodos funcionales (map, filter, reduce)
   - Tuplas
   - Sets y Maps
   - Objetos y tipos de objeto
   - Clases y herencia
   - Mixins y composición
   - Records y tipos mapeados

6. [Capítulo 6: Interfaces y Tipos Avanzados](06-interfaces.md)
   - Interfaces: definición e implementación
   - Type Aliases vs Interfaces
   - Intersection y Union types
   - Discriminated unions
   - Utility types (Partial, Required, Pick, Omit, Readonly)
   - Conditional types
   - Template literal types
   - Mapped types en profundidad
   - Branded types (tipos nominales)

7. [Capítulo 7: Asincronía y el Event Loop](07-asincronia.md)
   - El Event Loop: fases, microtasks y macrotasks
   - Callbacks y Callback Hell
   - Promesas: creación, encadenamiento, composición
   - Async/Await: sintaxis y patrones
   - Gestión de concurrencia (Promise.all, allSettled, race, any)
   - Timers (setTimeout, setInterval, setImmediate)
   - process.nextTick vs setImmediate
   - Workers Threads y child_process

8. [Capítulo 8: Manejo de Errores](08-manejo-de-errores.md)
   - try/catch/finally en profundidad
   - La clase Error y errores personalizados
   - Manejo de errores en promesas y async/await
   - Patrones de manejo de errores asíncronos
   - Error boundaries y manejo global
   - Errores en streams y eventos
   - Result pattern (never throw)
   - Winston y logging estructurado

9. [Capítulo 9: Módulos, Paquetes y npm](09-modulos-paquetes.md)
   - CommonJS vs ESM: historia y migración
   - package.json en profundidad
   - npm, yarn y pnpm
   - Versionado semántico
   - Publicar paquetes en npm
   - Monorepos con npm workspaces y Turborepo
   - Patrones de organización de proyectos
   - Path aliases con TypeScript

10. [Capítulo 10: Testing](10-testing.md)
    - Vitest y Jest: configuración y uso
    - Tests unitarios y de integración
    - Mocks, stubs, spies y fakes
    - Testing asíncrono
    - Snapshot testing
    - Code coverage
    - End-to-End con Playwright
    - Property-based testing con fast-check

11. [Capítulo 11: Generics en TypeScript](11-generics.md)
    - Funciones y tipos genéricos
    - Constraints y type bounds
    - Inferencia de tipos genéricos
    - Generics en clases e interfaces
    - Utility types genéricos avanzados
    - Patrones comunes con generics
    - Cuándo usar y cuándo no usar generics

12. [Capítulo 12: Buenas Prácticas](12-buenas-practicas.md)
    - Convenciones de nombrado
    - Organización de código y estructura de archivos
    - Principios SOLID en TypeScript
    - Clean Code aplicado a Node.js
    - ESLint, Prettier y Biome
    - tsconfig.json óptimo
    - Code Review checklist

13. [Capítulo 13: Patrones de Diseño en Node.js](13-patrones-de-diseno.md)
    - Patrones creacionales (Factory, Builder, Singleton, Dependency Injection)
    - Patrones estructurales (Adapter, Decorator, Facade, Proxy)
    - Patrones de comportamiento (Strategy, Observer, Command, Chain of Responsibility)
    - Patrones específicos de Node.js (Middleware, Revealing Module, Callback Pattern)

14. [Capítulo 14: Arquitectura Hexagonal y Domain-Driven Design](14-arquitectura-hexagonal.md)
    - Principios de la arquitectura hexagonal
    - Domain-Driven Design con TypeScript
    - Puertos y adaptadores
    - Value Objects, Entities y Aggregates
    - Implementación paso a paso
    - Proyecto completo de ejemplo

15. [Capítulo 15: Temas Avanzados](15-temas-avanzados.md)
    - Decorators experimentales y TC39
    - Metaprogramación con Reflect
    - Streams y Buffers
    - Performance: profiling, memory leaks, CPU profiling
    - Cluster module y PM2
    - Seguridad en Node.js
    - gRPC y Protobuf
    - Server-Sent Events y WebSockets nativos

16. [Capítulo 16: Desarrollo Web y APIs](16-desarrollo-web.md)
    - HTTP nativo en Node.js
    - Express.js en profundidad
    - Fastify: rendimiento y schema validation
    - NestJS: arquitectura opinionada
    - REST APIs con validación (Zod)
    - GraphQL con Apollo y TypeGraphQL
    - Bases de datos (Prisma, Drizzle, Knex)
    - Autenticación (JWT, OAuth2, Passport)
    - File uploads y streaming

17. [Capítulo 17: Ejercicios Prácticos](17-ejercicios.md)
    - Ejercicios por nivel (básico, intermedio, avanzado)
    - Proyectos integradores
    - Soluciones comentadas
