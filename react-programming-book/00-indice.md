# React: Guía Completa de Programación Moderna

Bienvenido al libro de **React moderno (v18/19+)**. Este libro ha sido diseñado para llevarte desde los conceptos fundamentales del renderizado declarativo y el Virtual DOM hasta el desarrollo de aplicaciones enterprise y la arquitectura basada en Server Components y Server Actions, utilizando las mejores prácticas de la industria con TypeScript y Vite.

## Índice General

1. [Capítulo 1: Introducción a React y el Virtual DOM](01-introduccion-y-virtual-dom.md)
   - Paradigmas de UI: declarativa vs. imperativa
   - Qué es React y por qué domina el frontend moderno
   - El Virtual DOM: ¿Mito o realidad? Funcionamiento y eficiencia
   - Configuración del entorno de desarrollo moderno (Node.js, Vite, npm)
   - Creación del primer proyecto React con TypeScript y Vite
   - Estructura y anatomía de un proyecto moderno

2. [Capítulo 2: JSX y el Motor de Reconciliación](02-jsx-y-reconciliacion.md)
   - JSX bajo la lupa: compilación de JSX a `React.createElement` / `_jsx`
   - Elementos de React vs. Componentes vs. Nodos del DOM real
   - El algoritmo de Reconciliación (Diffing Algorithm)
   - La importancia crítica del prop `key` en listas dinámicas
   - El compilador de React (React Compiler / React Forget)

3. [Capítulo 3: Estado, Props y Flujo de Datos Unidireccional](03-estado-y-ciclo-de-vida.md)
   - Props: datos inmutables y la analogía del pasaporte de viaje
   - El Estado (`useState`): la fuente de verdad mutable del componente
   - Asincronía en las actualizaciones y batches de estado (State Batching)
   - Flujo de datos descendente (Top-Down / Unidirectional Data Flow)
   - Elevación del estado (Lifting State Up) para sincronizar hermanos

4. [Capítulo 4: Ciclo de Vida y Efectos con useEffect](04-ciclo-de-vida-y-efectos.md)
   - El ciclo de vida de un componente: Montaje, Actualización y Desmontaje
   - Sincronización con APIs y servicios externos usando `useEffect`
   - Funciones de limpieza (Cleanup) y cómo prevenir memory leaks
   - La regla de oro: Efectos vs. Eventos (Cuándo usar `useEffect` y cuándo NO)
   - Dependencias de efectos y el bucle infinito de renders

5. [Capítulo 5: Hooks Fundamentales y Custom Hooks](05-hooks-y-custom-hooks.md)
   - Las reglas de los Hooks y el motor de ejecución secuencial interno
   - Memorización: `useMemo` y `useCallback` para prevenir re-cálculos costosos
   - Persistencia de datos mutables y acceso al DOM con `useRef`
   - Creación de Custom Hooks: abstracción de lógica reactiva y composición
   - Hooks experimentales y el ecosistema de utilidades de React

6. [Capítulo 6: Gestión del Estado Compartido con Context API](06-context-api.md)
   - El problema de Prop Drilling en arquitecturas profundas
   - Context API: Creación, provisión (`Provider`) y consumo (`useContext`)
   - Optimización de Context: cómo evitar renders innecesarios dividiendo contextos
   - Inyección de dependencias y servicios simulada con React Context
   - Contextos dinámicos y su combinación con custom hooks

7. [Capítulo 7: Gestión de Estado Avanzada con useReducer](07-usereducer.md)
   - Del estado disperso a máquinas de estado estructuradas
   - El patrón Reducer: Estado, Acciones y Reductores puros
   - Implementación de `useReducer` y comparación directa con `useState`
   - Combinación de `useReducer` con Context API para un gestor de estado global nativo
   - Reducers anidados y modularización de la lógica de negocio

8. [Capítulo 8: Gestión de Estado Global a Escala](08-estado-global.md)
   - Cuándo migrar a una librería de estado global y criterios de selección
   - Zustand: el estándar moderno simplificado basado en proxies y sin Boilerplate
   - Redux Toolkit (RTK): store, slices, actions, reducers y thunks asíncronos
   - Estado atómico con Jotai: atomización del estado y renderizado preciso
   - Comparativa de rendimiento y arquitectura de stores globales

9. [Capítulo 9: React 18/19: Características Concurrentes y Suspense](09-suspense-y-concurrencia.md)
   - React Concurrente y el renderizado no bloqueante (interrupción de renders)
   - Suspense: carga perezosa de componentes (`React.lazy`) y fetching declarativo
   - Transiciones de baja prioridad: `useTransition` y `useDeferredValue`
   - El hook `use` de React 19 para resolver promesas y contextos dinámicamente
   - Optimización de la experiencia de carga de datos e hidratación selectiva

10. [Capítulo 10: Server Components y React Server Actions](10-server-components-y-actions.md)
    - La arquitectura híbrida moderna: Client Components (`'use client'`) vs. Server Components
    - Ventajas de renderizar en el servidor: carga instantánea, menor JS en el cliente, SEO
    - React Server Actions: mutación de datos en servidor directamente desde componentes clientes
    - La directiva `'use server'` y validaciones de seguridad con formularios nativos
    - Streaming de HTML e hidratación progresiva

11. [Capítulo 11: Enrutamiento Moderno con React Router](11-react-router.md)
    - Single Page Applications y enrutamiento dinámico basado en componentes
    - Configuración moderna de rutas declarativas con React Router v6+
    - Rutas anidadas (`Outlet`), layouts compartidos y paso de parámetros dinámicos
    - Carga de datos optimizada: Data Loaders y Actions para fetching pre-renderizado
    - Rutas protegidas y guards funcionales de autenticación

12. [Capítulo 12: Formularios Avanzados y Validaciones](12-formularios-y-validaciones.md)
    - Componentes controlados frente a no controlados
    - Formularios de alto rendimiento y fácil mantenimiento con React Hook Form
    - Validación de esquemas estructurados con Zod
    - Gestión de estados de carga, errores de red y retroalimentación interactiva
    - Formularios multi-paso complejos con persistencia de datos

13. [Capítulo 13: Optimización de Rendimiento y Renderizado](13-optimizacion-y-rendimiento.md)
    - El ciclo de renderizado de React: cuándo renderiza un componente y cómo evitarlo
    - Evitar re-renders inútiles con `React.memo`
    - Virtualización de listas masivas con `react-virtual` o `react-window`
    - Code splitting, dynamic imports y lazy loading estratégico
    - Perfilado de aplicaciones paso a paso con las React Developer Tools

14. [Capítulo 14: Testing en React: Unitario, Integración y E2E](14-testing.md)
    - Filosofía de testing en el frontend: qué testear para maximizar el ROI de desarrollo
    - Configuración del entorno de testing moderno: Vitest y React Testing Library
    - Testing de componentes interactivos y custom hooks
    - Mocking de llamadas de red asíncronas con MSW (Mock Service Worker)
    - Pruebas de extremo a extremo (E2E) robustas con Playwright

15. [Capítulo 15: Proyecto Práctico Completo e Integrador](15-proyecto-practico.md)
    - Definición del proyecto: Aplicación E-Commerce de alto rendimiento
    - Estructura de carpetas Feature-driven profesional para escalabilidad empresarial
    - Construcción interactiva del catálogo, carrito de compras (Zustand) y checkout (React Hook Form)
    - Optimización y testing de integración global
    - Despliegue automatizado y buenas prácticas para entornos productivos
