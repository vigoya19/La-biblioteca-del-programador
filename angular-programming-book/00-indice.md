# Angular: Guía Completa de Programación Moderna

Bienvenido al libro de **Angular moderno (v17/18/19+)**. Este libro ha sido diseñado para llevarte desde los fundamentos más básicos hasta el diseño y la optimización de aplicaciones a nivel empresarial, utilizando las prácticas más actuales del framework, tales como componentes Standalone, Signals, Control Flow nativo, vistas diferibles, zoneless, micro frontends y PWAs.

## Índice General

1. [Capítulo 1: Introducción a Angular](01-introduccion.md)
   - ¿Qué es Angular y por qué elegirlo para desarrollo empresarial?
   - Evolución del framework: de AngularJS al Angular moderno standalone
   - Filosofía de diseño: tipado fuerte (TypeScript), modularidad moderna y arquitectura basada en componentes
   - Configuración del entorno de desarrollo (Node.js, npm, Angular CLI)
   - Creación del primer proyecto Standalone (`ng new` con `--standalone`)
   - Anatomía de un proyecto de Angular moderno

2. [Capítulo 2: Arquitectura y Componentes](02-arquitectura-y-componentes.md)
   - ¿Qué es un Componente Standalone?
   - Metadatos del decorador `@Component` (`selector`, `templateUrl`, `styleUrls`, `imports`)
   - Ciclo de vida y compilación de componentes
   - Creación de componentes con Angular CLI (`ng generate component`)
   - Buenas prácticas para la organización y cohesión de componentes

3. [Capítulo 3: Templates y Control Flow Sintáctico](03-templates-y-directivas.md)
   - Sintaxis de plantillas (interpolation, property binding, event binding, two-way binding)
   - Atributo HTML vs. Propiedad DOM: la diferencia crítica que el 90% de desarrolladores ignora
   - El nuevo Control Flow sintáctico (`@if`, `@else`, `@for`, `@switch`)
   - Internals del compilador: cómo Ivy transforma `@for` en instrucciones de renderizado optimizadas
   - Pipes puros e impuros: rendimiento, memorización y cuándo crear pipes personalizados
   - Custom Directives: Directivas de Atributo, Directivas Estructurales y `HostDirectives`
   - La API `model()` para Two-Way Binding nativo sin `ngModel`
   - Signal Queries: `viewChild()`, `viewChildren()`, `contentChild()`, `contentChildren()`
   - Proyección de Contenido (`ng-content`), `@ContentChild` y proyección condicional
   - Sanitización de HTML dinámico y prevención de ataques XSS con `DomSanitizer`

4. [Capítulo 4: Angular Signals — El Nuevo Motor Reactivo](04-signals.md)
   - La teoría de la Reactividad Fina (Fine-Grained Reactivity) y el grafo de dependencias push/pull
   - Writable Signals: `signal()`, `set()`, `update()`, igualdad personalizada con `equal`
   - Computed Signals: memorización, evaluación perezosa y rastreo dinámico de dependencias
   - Effects: reglas de ejecución, `onCleanup`, `AbortController` y `untracked()`
   - Signal Queries: `viewChild()`, `viewChildren()`, `contentChild()`, `contentChildren()`
   - `model()` y `linkedSignal()`: Two-Way Binding reactivo y derivación con reset automático
   - Resource API y `rxResource()`: fetching asíncrono nativo basado en Signals
   - Interoperabilidad entre Signals y RxJS (`toSignal`, `toObservable`)

5. [Capítulo 5: Servicios e Inyección de Dependencias](05-servicios-e-inyeccion-de-dependencias.md)
   - El patrón de Inyección de Dependencias (DI) en Angular
   - Creación de servicios con `@Injectable`
   - El decorador `@Inject` clásico frente a la nueva función **`inject()`**
   - Jerarquía completa de inyectores: `EnvironmentInjector`, `ElementInjector`, `NodeInjector`
   - Proveedores avanzados: `useClass`, `useFactory`, `useExisting` y `useValue`
   - Multi-providers y `InjectionToken` con tipado genérico
   - `runInInjectionContext()` y contextos de inyección dinámicos

6. [Capítulo 6: Formularios y Validaciones](06-formularios-y-validaciones.md)
   - Template-driven Forms frente a Reactive Forms
   - Strictly Typed Forms con `NonNullableFormBuilder`
   - `FormArray` para colecciones dinámicas
   - Validadores síncronos, asíncronos y cross-field personalizados
   - Formularios dinámicos desde metadatos JSON
   - Reactividad en formularios con Signals y RxJS
   - Accesibilidad (a11y) en formularios: `aria-*` y live regions

7. [Capítulo 7: Enrutamiento y Navegación](07-enrutamiento-y-navegacion.md)
   - Configuración moderna con `provideRouter` y sus features
   - Lazy Loading: `loadComponent` y `loadChildren`
   - Component Input Binding con `withComponentInputBinding()`
   - Guards funcionales: `CanActivateFn`, `CanDeactivateFn`, `CanMatchFn`
   - Resolvers funcionales para pre-carga de datos
   - Estrategias de precarga y View Transitions API

8. [Capítulo 8: Comunicación Asíncrona y HttpClient](08-comunicacion-asincrona-http.md)
   - Configuración moderna con `provideHttpClient`
   - Operaciones CRUD con tipado estricto
   - Operadores RxJS: `switchMap`, `exhaustMap`, `concatMap`, `mergeMap`
   - Interceptores HTTP funcionales: JWT, refresh concurrente, errores globales
   - Cancelación con `takeUntilDestroyed` y `AbortController`
   - Upload/download con progreso y WebSockets

9. [Capítulo 9: Gestión de Estado](09-gestion-de-estado.md)
   - Cuándo usar estado local, servicios compartidos o Store global
   - Patrón de State Service con Signals (ligero, sin dependencias externas)
   - NgRx Signals Store: `signalStore`, `withState`, `withComputed`, `withMethods`
   - Custom Store Features reutilizables: `withLoading()`, `withPagination()`, `withLocalStorage()`
   - Entities Management con `withEntities()` de `@ngrx/signals/entities`
   - Comparativa de alternativas: NgRx clásico, Elf, StateAdapt y cuándo usar cada una
   - DevTools y depuración de estado

10. [Capítulo 10: Ciclo de Vida y Optimización de Rendimiento](10-ciclo-de-vida-y-optimizaciones.md)
    - Hooks del ciclo de vida en profundidad
    - `afterNextRender()` y `afterRender()`: hooks específicos para el browser post-SSR
    - `DestroyRef` y `takeUntilDestroyed()`: la solución definitiva al memory leak
    - Estrategias de detección de cambios: `Default` vs `OnPush`
    - Zoneless Angular: aplicaciones sin `zone.js`
    - Perfilado con Angular DevTools y Lighthouse
    - `NgOptimizedImage`, Web Workers y budget de bundles

11. [Capítulo 11: Vistas Diferibles (@defer) en Profundidad](11-vistas-diferibles.md)
    - Internals del compilador y code-splitting automático
    - Disparadores: `idle`, `immediate`, `timer`, `viewport`, `interaction`, `hover`
    - Disparadores personalizados con `when` y Signals
    - Bloques `@placeholder`, `@loading`, `@error` con timing avanzado
    - Prefetching y composición con `@for` para listas infinitas
    - Análisis de bundle con `source-map-explorer`

12. [Capítulo 12: Server-Side Rendering (SSR), SSG e Hidratación](12-ssr-ssg-y-hidratacion.md)
    - SSR con `@angular/ssr` y Express engine
    - Full Client-Side Hydration libre de parpadeos
    - Hidratación incremental y parcial (experimental)
    - SSG/Prerendering de rutas estáticas y dinámicas
    - Transfer State API para evitar peticiones duplicadas
    - Metadatos SEO dinámicos con `Meta` y `Title` services
    - Despliegue SSR: Firebase, Vercel, AWS Lambda y Docker

13. [Capítulo 13: Testing en Angular](13-testing-en-angular.md)
    - Filosofía de testing: pirámide, ROI y qué testear primero
    - Configuración de Vitest para Angular (migración paso a paso desde Karma)
    - TDD guiado: desarrollo dirigido por tests con ejemplo completo paso a paso
    - Testing de servicios con `HttpTestingController` (ejercicio práctico)
    - Testing de componentes standalone con `TestBed` (ejercicio práctico)
    - Testing de Signals, Computeds y Effects
    - Testing asíncrono: `fakeAsync`, `tick`, `flush`
    - Testing de guards, resolvers y pipes de forma aislada
    - Cobertura de código: umbrales mínimos y reporting en CI/CD
    - Pruebas E2E con Playwright: setup, Page Objects, ejercicio paso a paso

14. [Capítulo 14: Buenas Prácticas y Patrones de Diseño](14-buenas-practicas-y-patrones.md)
    - Principios SOLID aplicados al frontend
    - Estructura de carpetas Feature-driven y Domain-driven
    - Smart vs Dumb Components: reglas estrictas y excepciones
    - Composición sobre herencia: `hostDirectives` y mixins funcionales
    - Monorepos con Nx: librerías compartidas y affected commands
    - Code Review Checklist exhaustiva

15. [Capítulo 15: Proyecto Práctico Completo e Integrador](15-ejercicios-practicos.md)
    - Definición del proyecto: Aplicación E-Commerce de alto rendimiento
    - Ejercicio 1: Andamiaje del proyecto y diseño modular de rutas
    - Ejercicio 2: Construcción del catálogo con `@for`, `track` y filtros reactivos
    - Ejercicio 3: Carrito de compras con NgRx Signals Store paso a paso
    - Ejercicio 4: Formulario de checkout con validaciones avanzadas
    - Ejercicio 5: Optimización con `@defer` y análisis de bundle
    - Ejercicio 6: Escritura de tests unitarios y E2E para el proyecto
    - Ejercicio 7: SSR, SEO y despliegue a producción
    - Retos adicionales para el lector

16. [Capítulo 16: Micro Frontends y Arquitecturas de Múltiples Equipos](16-microfrontends.md)
    - ¿Cuándo aplicar Micro Frontends y cuándo NO?
    - Webpack Module Federation: Shell, Remotos y shared dependencies
    - Native Federation con Esbuild e Import Maps
    - Comunicación inter-MFE: Custom Events, servicios compartidos y contratos
    - Resiliencia, fallbacks y SSR en arquitecturas federadas
    - Monorepos con Nx y CI/CD para MFEs
    - Caso de estudio: migración progresiva de un monolito a MFEs

17. [Capítulo 17: Angular Material, CDK y Animaciones](17-material-cdk-animaciones.md)
    - Angular Material: filosofía, temas personalizados y diseño responsivo
    - Los componentes más utilizados: Tables, Dialogs, Snackbars, Forms, Navigation
    - Angular CDK: herramientas headless para construir UI a medida
    - CDK esenciales: Overlay, Drag & Drop, Virtual Scrolling, A11y, Clipboard
    - Sistema de Animaciones de Angular: `trigger`, `state`, `transition`, `animate`
    - Animaciones de entrada/salida, listas animadas y animaciones de ruta
    - Integración de animaciones con el Control Flow (`@if`, `@for`)
    - Rendimiento de animaciones: `will-change`, GPU compositing y `requestAnimationFrame`

18. [Capítulo 18: Internacionalización (i18n) y Accesibilidad (a11y)](18-i18n-y-a11y.md)
    - Angular i18n nativo: marcado de textos, extracción y archivos XLIFF/XMB
    - Compilación multi-idioma: una build por locale vs. runtime translation
    - Alternativas de i18n en runtime: `@ngx-translate` y `Transloco`
    - Formateo localizado de fechas, números y monedas con `DatePipe`, `DecimalPipe`, `CurrencyPipe`
    - Pluralización y selección de género con ICU Message Format
    - Accesibilidad (a11y): principios WCAG 2.1, roles ARIA, focus management
    - CDK A11y: `FocusTrap`, `LiveAnnouncer`, `FocusMonitor`, navegación por teclado
    - Auditoría de accesibilidad con Lighthouse, axe-core y `eslint-plugin-jsx-a11y`

19. [Capítulo 19: Progressive Web Apps (PWAs)](19-pwas.md)
    - ¿Qué es una PWA y por qué Angular es ideal para construirlas?
    - Configuración con `@angular/pwa`: Service Worker, manifest y estrategias de caché
    - Estrategias de caché: `performance` (prefetch) vs. `freshness` (network-first)
    - Actualizaciones de la aplicación: `SwUpdate`, versionado y notificaciones al usuario
    - Push Notifications con `SwPush` y Web Push API
    - Soporte offline: interceptores de caché, sincronización en background
    - Instalación de la PWA: `beforeinstallprompt`, UX de instalación personalizada
    - Auditoría PWA con Lighthouse y criterios de instalabilidad
