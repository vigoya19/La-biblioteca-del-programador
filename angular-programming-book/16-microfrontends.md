# Capítulo 16: Micro Frontends y Arquitecturas de Múltiples Equipos

> "El objetivo de una arquitectura de Micro Frontends no es dividir tu código, sino dividir tu organización para que cada equipo pueda entregar valor de negocio de forma independiente."

En los ecosistemas empresariales modernos, las aplicaciones web han dejado de ser proyectos individuales gestionados por un único equipo de frontend. Las grandes plataformas digitales (banca online, marketplaces, portales corporativos) involucran decenas de equipos de desarrollo trabajando simultáneamente sobre dominios de negocio distintos: el equipo de pagos, el equipo de catálogo, el equipo de logística, el equipo de administración.

Cuando todos estos equipos trabajan sobre un único monolito Angular, los conflictos de merge, los tiempos de build que se disparan a 20+ minutos, las dependencias cruzadas incontrolables y los despliegues que requieren coordinar a toda la organización se convierten en un cuello de botella insostenible que frena la innovación.

La arquitectura de **Micro Frontends (MFE)** resuelve exactamente este problema. Al igual que los microservicios en el backend descomponen un servidor monolítico en servicios independientes desplegables, los Micro Frontends descomponen una aplicación web monolítica en **sub-aplicaciones frontend autónomas** que se integran de forma transparente en una experiencia de usuario unificada.

En este capítulo, exploraremos la teoría fundacional de los Micro Frontends, analizaremos en profundidad las dos tecnologías dominantes del ecosistema Angular (**Webpack Module Federation** y **Native Federation con Esbuild**), y construiremos una arquitectura federada completa desde cero con comunicación inter-MFE, routing federado, gestión de dependencias compartidas, estrategias de resiliencia ante caídas y consideraciones avanzadas de SSR.

---

## 16.1 ¿Qué son los Micro Frontends y Cuándo Aplicar esta Arquitectura?

Un **Micro Frontend** es una porción vertical de una aplicación web que cubre un dominio de negocio completo (desde la UI hasta la conexión con las APIs de ese dominio), desarrollada, testeada y desplegada de forma completamente independiente por un equipo autónomo.

```
┌──────────────────────────────────────────────────────────────────────┐
│                        Experiencia de Usuario Unificada              │
│                                                                      │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────────┐  │
│  │   MFE Catálogo  │  │   MFE Carrito   │  │  MFE Administración │  │
│  │   (Equipo Alpha) │  │  (Equipo Beta)  │  │   (Equipo Gamma)   │  │
│  │                 │  │                 │  │                     │  │
│  │  Angular v19    │  │  Angular v18    │  │   Angular v19       │  │
│  │  Standalone     │  │  Standalone     │  │   + NgRx Signals    │  │
│  │  Despliega lunes│  │ Despliega viernes│  │   Despliega diario  │  │
│  └─────────────────┘  └─────────────────┘  └─────────────────────┘  │
│                                                                      │
│                    ┌────────────────────────┐                        │
│                    │  Shell / Host Angular  │                        │
│                    │  (Equipo Plataforma)   │                        │
│                    │  Routing + Layout      │                        │
│                    └────────────────────────┘                        │
└──────────────────────────────────────────────────────────────────────┘
```

### Principios Fundamentales de los Micro Frontends

1. **Autonomía de Equipo**: Cada equipo posee su propio repositorio (o carpeta en un monorepo), su propio pipeline de CI/CD, sus propias dependencias y su propio calendario de releases. No existe coordinación obligatoria para desplegar.
2. **Aislamiento Tecnológico**: Aunque en este libro nos centramos en Angular, la arquitectura permite que cada MFE utilice la versión del framework que necesite. Un MFE puede estar en Angular v18 mientras otro usa v19, e incluso en teorías agnósticas un MFE podría ser React o Vue (aunque no es lo recomendable por coherencia de UX).
3. **Integración en Tiempo de Ejecución**: A diferencia de una librería compartida en un monorepo (que se integra en tiempo de compilación), los Micro Frontends se integran en **runtime**. El Shell descarga dinámicamente el código del MFE remoto desde un servidor CDN y lo monta en el DOM.
4. **Resilencia y Tolerancia a Fallos**: Si el servicio del MFE de "Recomendaciones de Productos" se cae, la aplicación principal debe seguir funcionando. El shell debe renderizar un fallback elegante.

### ¿Cuándo NO usar Micro Frontends?

La complejidad operacional de los MFE no es trivial. **No apliques esta arquitectura si:**
* Tu equipo tiene menos de 15-20 desarrolladores.
* Tu aplicación no tiene dominios de negocio claramente delimitados e independientes.
* Tus tiempos de build son razonables (< 5 minutos) y no tienes conflictos frecuentes de merge.
* No tienes la infraestructura de DevOps necesaria (múltiples pipelines, CDN, registros de versiones).

> **Regla de Oro**: Si puedes resolver tu problema con un monorepo bien estructurado con Nx y librerías compartidas, esa es siempre la solución más simple y preferida. Los Micro Frontends son para cuando el monolito colapsa organizacionalmente.

---

## 16.2 Estrategias de Integración: Build-time vs. Runtime

Existen dos grandes familias de integración de Micro Frontends:

| Estrategia | Mecanismo | Pros | Contras |
|---|---|---|---|
| **Build-time** (npm packages) | Cada MFE se publica como un paquete npm y se instala como dependencia del shell | Máximo rendimiento, Tree-Shaking completo | Acoplamiento en build, coordinación de versiones |
| **Runtime** (Module Federation) | Cada MFE se despliega independientemente y se carga dinámicamente en el browser | Despliegue 100% independiente, versionado libre | Overhead de red, complejidad de configuración |

En este capítulo nos centramos en la **integración en runtime**, que es la que verdaderamente desacopla los ciclos de vida de los equipos.

---

## 16.3 Webpack Module Federation: La Tecnología Fundacional

**Module Federation** es un plugin del bundler Webpack (introducido en Webpack 5) que permite a múltiples aplicaciones compiladas de forma independiente compartir módulos JavaScript entre sí en tiempo de ejecución.

### Conceptos Clave de Module Federation

```
┌──────────────────────────────────────────────────────────────────┐
│                       Module Federation                          │
│                                                                  │
│  Host (Shell)                   Remote (MFE)                     │
│  ┌────────────────────────┐    ┌──────────────────────────────┐  │
│  │ - Consume módulos      │◄───│ - Expone módulos              │  │
│  │   remotos vía          │    │   (componentes, rutas)        │  │
│  │   loadRemoteModule()   │    │ - Publica un remoteEntry.js   │  │
│  │ - Define shared deps   │    │ - Define shared deps          │  │
│  └────────────────────────┘    └──────────────────────────────┘  │
│                                                                  │
│  Shared Dependencies (singleton)                                 │
│  ┌──────────────────────────────────────────────────────────────┐│
│  │  @angular/core, @angular/router, rxjs, zone.js               ││
│  │  (Se cargan una sola vez en memoria compartida)               ││
│  └──────────────────────────────────────────────────────────────┘│
└──────────────────────────────────────────────────────────────────┘
```

* **Host (Shell)**: La aplicación principal que orquesta la carga de los MFEs remotos. Define el layout global, la barra de navegación, la autenticación y el routing principal.
* **Remote (MFE)**: Cada sub-aplicación que expone componentes, rutas o módulos para ser consumidos por el Host. Cada Remote publica un archivo manifiesto llamado `remoteEntry.js` que el Host descarga para saber qué módulos están disponibles.
* **Shared Dependencies**: Librerías comunes (como `@angular/core`, `@angular/router`, `rxjs`) que se configuran para cargarse una única vez en memoria y compartirse entre el Host y todos los Remotes, evitando duplicación de código.

### Implementación con `@angular-architects/module-federation`

La librería `@angular-architects/module-federation` del reconocido GDE Manfred Steyer simplifica enormemente la configuración de Module Federation en proyectos Angular.

#### Paso 1: Instalación en el Shell (Host)

```bash
# En el proyecto del Shell
ng add @angular-architects/module-federation --project shell --port 4200 --type host
```

#### Paso 2: Instalación en el MFE Remoto

```bash
# En el proyecto del MFE de Catálogo
ng add @angular-architects/module-federation --project mfe-catalogo --port 4201 --type remote
```

#### Paso 3: Configuración del Remote — `webpack.config.js`

El MFE de Catálogo expone sus rutas standalone como un módulo federado:

```javascript
// mfe-catalogo/webpack.config.js
const { shareAll, withModuleFederationPlugin } = require("@angular-architects/module-federation/webpack");

module.exports = withModuleFederationPlugin({
  name: "mfeCatalogo",
  
  // Módulos que este Remote EXPONE al mundo exterior
  exposes: {
    // La clave es el alias público, el valor es la ruta física al archivo
    "./routes": "./src/app/catalogo.routes.ts",
    "./CatalogoComponent": "./src/app/catalogo/catalogo.component.ts"
  },

  // Dependencias compartidas para evitar duplicación en memoria
  shared: {
    ...shareAll({
      singleton: true,        // Una única instancia global compartida
      strictVersion: true,     // Error si las versiones no son compatibles
      requiredVersion: "auto"  // Detecta la versión del package.json automáticamente
    })
  }
});
```

#### Paso 4: Configuración del Host — `webpack.config.js`

El Shell declara qué remotos existen y dónde encontrar sus manifiestos:

```javascript
// shell/webpack.config.js
const { shareAll, withModuleFederationPlugin } = require("@angular-architects/module-federation/webpack");

module.exports = withModuleFederationPlugin({
  remotes: {
    // El nombre DEBE coincidir con el "name" del webpack.config del Remote
    mfeCatalogo: "http://localhost:4201/remoteEntry.js"
  },

  shared: {
    ...shareAll({
      singleton: true,
      strictVersion: true,
      requiredVersion: "auto"
    })
  }
});
```

#### Paso 5: Integración en el Routing del Shell

```typescript
// shell/src/app/app.routes.ts
import { Routes } from "@angular/router";
import { loadRemoteModule } from "@angular-architects/module-federation";

export const routes: Routes = [
  { path: "", redirectTo: "home", pathMatch: "full" },
  
  { 
    path: "home", 
    loadComponent: () => import("./home/home.component").then(m => m.HomeComponent) 
  },
  
  // Carga perezosa del MFE de Catálogo desde el servidor remoto
  {
    path: "catalogo",
    loadChildren: () => loadRemoteModule({
      type: "module",
      remoteEntry: "http://localhost:4201/remoteEntry.js",
      exposedModule: "./routes"
    }).then((m) => m.catalogoRoutes)
  },

  // Ruta de fallback si el MFE no se encuentra
  { path: "**", loadComponent: () => import("./not-found.component").then(m => m.NotFoundComponent) }
];
```

---

## 16.4 Native Federation: La Alternativa Moderna sin Webpack

A partir de Angular v17+, el builder por defecto del CLI migró de Webpack a **Esbuild + Vite**, lo cual es espectacularmente más rápido para compilación y hot-reload. Sin embargo, Esbuild **no soporta nativamente** Webpack Module Federation.

Para resolver este problema, Manfred Steyer creó **`@angular-architects/native-federation`**, una implementación de Module Federation construida sobre los **Import Maps** nativos del navegador y el **ES Module Loader** estándar de JavaScript, eliminando completamente la dependencia de Webpack.

### ¿Por qué elegir Native Federation sobre Webpack Module Federation?

| Característica | Webpack Module Federation | Native Federation |
|---|---|---|
| **Builder de Angular** | Requiere Webpack (builder clásico) | Compatible con Esbuild (builder moderno) |
| **Velocidad de Build** | Lenta (Webpack) | Ultra rápida (Esbuild) |
| **Estándar Web** | Propietario de Webpack | Basado en Import Maps (estándar W3C) |
| **Hot Module Replacement** | Lento | Instantáneo (Vite) |
| **Complejidad de Config** | Media-Alta | Media |
| **Madurez** | Alta (desde 2020) | Media (desde 2023) |

### Implementación con Native Federation

#### Paso 1: Instalación

```bash
# En el Shell (Host)
ng add @angular-architects/native-federation --project shell --port 4200 --type dynamic-host

# En el Remote (MFE Catálogo)
ng add @angular-architects/native-federation --project mfe-catalogo --port 4201 --type remote
```

#### Paso 2: Configuración del Remote — `federation.config.js`

```javascript
// mfe-catalogo/federation.config.js
const { withNativeFederation, shareAll } = require("@angular-architects/native-federation/config");

module.exports = withNativeFederation({
  name: "mfeCatalogo",
  
  exposes: {
    "./routes": "./src/app/catalogo.routes.ts"
  },

  shared: {
    ...shareAll({
      singleton: true,
      strictVersion: true,
      requiredVersion: "auto"
    })
  },

  // Opcional: Definir que las dependencias de Angular se omitan
  // del bundle del Remote ya que el Host las proveerá
  skip: [
    "rxjs/ajax",
    "rxjs/fetch",
    "rxjs/testing",
    "rxjs/webSocket"
  ]
});
```

#### Paso 3: Configuración Dinámica del Host — `federation.manifest.json`

En lugar de hardcodear las URLs de los remotos en el código, Native Federation permite definir un **manifiesto dinámico** que se carga en runtime. Esto permite cambiar las URLs de los MFEs sin recompilar el Shell:

```json
{
  "mfeCatalogo": "http://localhost:4201/remoteEntry.json",
  "mfeCarrito": "http://localhost:4202/remoteEntry.json",
  "mfeAdmin": "http://cdn.empresa.com/admin/v2.3.1/remoteEntry.json"
}
```

#### Paso 4: Inicialización del Shell con el Manifiesto

```typescript
// shell/src/main.ts
import { initFederation } from "@angular-architects/native-federation";

initFederation("assets/federation.manifest.json")
  .catch((err) => console.error("Error inicializando Native Federation:", err))
  .then(() => import("./bootstrap"))
  .catch((err) => console.error("Error arrancando la aplicación Shell:", err));
```

```typescript
// shell/src/bootstrap.ts
import { bootstrapApplication } from "@angular/platform-browser";
import { appConfig } from "./app/app.config";
import { AppComponent } from "./app/app.component";

bootstrapApplication(AppComponent, appConfig)
  .catch((err) => console.error(err));
```

#### Paso 5: Routing Federado en el Shell

```typescript
// shell/src/app/app.routes.ts
import { Routes } from "@angular/router";
import { loadRemoteModule } from "@angular-architects/native-federation";

export const routes: Routes = [
  {
    path: "catalogo",
    loadChildren: () => loadRemoteModule("mfeCatalogo", "./routes")
      .then((m) => m.catalogoRoutes)
  },
  {
    path: "carrito",
    loadChildren: () => loadRemoteModule("mfeCarrito", "./routes")
      .then((m) => m.carritoRoutes)
  }
];
```

---

## 16.5 Comunicación entre Micro Frontends

Uno de los desafíos más críticos y delicados en una arquitectura de Micro Frontends es la **comunicación inter-MFE**. Cuando el usuario agrega un producto al carrito desde el MFE de Catálogo, ¿cómo se entera el MFE del Carrito (que es una aplicación completamente separada) de que se ha añadido un nuevo artículo?

Existen tres patrones principales, cada uno con sus ventajas y riesgos:

### Patrón 1: Custom Events del DOM (Recomendado para Baja Complejidad)

Los **Custom Events** nativos del navegador son la forma más desacoplada y agnóstica al framework de comunicar MFEs. Cualquier MFE puede emitir un evento en el `window` global y cualquier otro MFE puede escucharlo.

```typescript
// MFE Catálogo: Emitir un evento cuando el usuario agrega un producto
export function emitirProductoAgregado(producto: { id: string; nombre: string; precio: number }): void {
  const evento = new CustomEvent("mfe:producto-agregado", {
    detail: producto,
    bubbles: true,   // Propaga el evento hacia arriba en el árbol del DOM
    composed: true    // Atraviesa los Shadow DOM boundaries (si los hubiere)
  });
  
  window.dispatchEvent(evento);
}
```

```typescript
// MFE Carrito: Escuchar el evento global
import { Component, OnInit, OnDestroy, signal } from "@angular/core";

@Component({
  selector: "app-carrito-widget",
  standalone: true,
  template: `<span class="badge">{{ cantidadItems() }}</span>`
})
export class CarritoWidgetComponent implements OnInit, OnDestroy {
  cantidadItems = signal(0);
  
  private readonly handler = (event: Event) => {
    const customEvent = event as CustomEvent;
    console.log("Producto recibido desde MFE Catálogo:", customEvent.detail);
    this.cantidadItems.update(c => c + 1);
  };

  ngOnInit(): void {
    window.addEventListener("mfe:producto-agregado", this.handler);
  }

  ngOnDestroy(): void {
    window.removeEventListener("mfe:producto-agregado", this.handler);
  }
}
```

### Patrón 2: Servicio Compartido vía Inyección de Dependencias (Recomendado para Angular-Angular)

Cuando todos los MFEs son Angular, podemos crear un **servicio singleton compartido** que viva en una librería común importada tanto por el Shell como por los Remotos. Al configurar la librería como `shared: { singleton: true }` en la configuración de federación, Angular garantiza que todos los MFEs reciban exactamente la misma instancia del servicio.

```typescript
// shared-lib/src/lib/mfe-communication.service.ts
import { Injectable, signal, computed } from "@angular/core";

export interface ProductoCarrito {
  id: string;
  nombre: string;
  precio: number;
  cantidad: number;
}

@Injectable({ providedIn: "root" })
export class MfeCommunicationService {
  private readonly _items = signal<ProductoCarrito[]>([]);
  
  readonly items = this._items.asReadonly();
  readonly totalItems = computed(() => this._items().reduce((acc, i) => acc + i.cantidad, 0));
  readonly totalPrecio = computed(() => this._items().reduce((acc, i) => acc + i.precio * i.cantidad, 0));

  agregarProducto(producto: Omit<ProductoCarrito, "cantidad">): void {
    this._items.update(items => {
      const existente = items.find(i => i.id === producto.id);
      if (existente) {
        return items.map(i => i.id === producto.id ? { ...i, cantidad: i.cantidad + 1 } : i);
      }
      return [...items, { ...producto, cantidad: 1 }];
    });
  }

  eliminarProducto(id: string): void {
    this._items.update(items => items.filter(i => i.id !== id));
  }
}
```

> **Advertencia crítica**: Para que el servicio compartido funcione como singleton real entre MFEs, la librería que lo contiene DEBE estar declarada como `shared` con `singleton: true` en la configuración de federación tanto del Host como de todos los Remotos. Si no se configura correctamente, cada MFE creará su propia instancia independiente del servicio y los datos no se sincronizarán.

### Patrón 3: Estado Global vía `window` (Último Recurso)

En escenarios donde los MFEs utilizan frameworks diferentes (Angular + React + Vue), el objeto global `window` puede actuar como bus de estado compartido. **Este patrón debe usarse con extrema cautela** y con un contrato de tipos bien definido:

```typescript
// Contrato de tipos compartido entre todos los MFEs
declare global {
  interface Window {
    __MFE_STATE__: {
      auth: { token: string; userId: string } | null;
      tema: "light" | "dark";
    };
  }
}

// Inicialización en el Shell
window.__MFE_STATE__ = {
  auth: null,
  tema: "light"
};

// Lectura desde cualquier MFE
const token = window.__MFE_STATE__?.auth?.token;
```

---

## 16.6 Gestión de Dependencias Compartidas y Conflictos de Versión

Uno de los mayores peligros en una arquitectura federada es la **duplicación de dependencias** en memoria. Si el Shell carga Angular v19 y el MFE de Catálogo también empaqueta su propia copia de Angular v19, el navegador del usuario descargará Angular dos veces, duplicando el consumo de memoria y, peor aún, generando errores fatales de inyección de dependencias (ya que habrá dos instancias diferentes del inyector raíz).

### Configuración Correcta de Shared Dependencies

```javascript
// federation.config.js (tanto en Host como en cada Remote)
shared: {
  "@angular/core": { singleton: true, strictVersion: true, requiredVersion: "^19.0.0" },
  "@angular/common": { singleton: true, strictVersion: true, requiredVersion: "^19.0.0" },
  "@angular/router": { singleton: true, strictVersion: true, requiredVersion: "^19.0.0" },
  "@angular/forms": { singleton: true, strictVersion: true, requiredVersion: "^19.0.0" },
  "rxjs": { singleton: true, strictVersion: false, requiredVersion: ">=7.0.0" },
  "zone.js": { singleton: true, strictVersion: true, requiredVersion: "^0.14.0" },
  
  // Librería compartida interna de la empresa
  "@empresa/shared-ui": { singleton: true, strictVersion: true, requiredVersion: "^2.0.0" },
  "@empresa/shared-auth": { singleton: true, strictVersion: true, requiredVersion: "^1.5.0" }
}
```

### ¿Qué pasa cuando las versiones no coinciden?

Cuando `strictVersion: true` está activado y el Host tiene Angular `19.0.0` pero el Remote fue compilado con Angular `19.2.0`:

1. **Si las versiones son compatibles semánticamente** (minor/patch differ): Module Federation usa la versión del Host y la comparte con el Remote. Funciona correctamente.
2. **Si hay un major version mismatch** (ej. Host v18 vs Remote v19): Module Federation lanza un **error en tiempo de ejecución** en la consola del navegador, indicando que las versiones son incompatibles. El Remote no se cargará.
3. **Si `strictVersion: false`**: Module Federation intentará usar cualquier versión disponible. Peligroso pero útil para librerías tolerantes como `rxjs` o `lodash`.

### La Estrategia de Versionado Recomendada para Empresas

```
┌─────────────────────────────────────────────────────────────────┐
│               Estrategia de Actualización Angular               │
│                                                                 │
│  1. El equipo de Plataforma (Shell) actualiza Angular           │
│  2. Publica una nueva versión de la librería @empresa/shared-*  │
│  3. Cada equipo de MFE actualiza en su propio ritmo             │
│     (ventana máxima: 2 sprints / 4 semanas)                     │
│  4. El Shell mantiene compatibilidad con N-1 versión            │
└─────────────────────────────────────────────────────────────────┘
```

---

## 16.7 Resiliencia y Fallbacks: Manejo de Errores en MFEs

En producción, los MFEs remotos pueden no estar disponibles por múltiples razones: un despliegue fallido, un CDN caído, un error en el pipeline de CI. El Shell **debe** manejar estas situaciones de forma elegante sin que toda la aplicación colapse.

### Implementación de un Error Boundary para MFEs

```typescript
// shell/src/app/core/components/mfe-error-boundary.component.ts
import { Component, input } from "@angular/core";

@Component({
  selector: "app-mfe-error-boundary",
  standalone: true,
  template: `
    <div class="p-8 bg-amber-50 border border-amber-200 rounded-lg text-center">
      <div class="text-4xl mb-4">⚠️</div>
      <h2 class="text-xl font-bold text-amber-800 mb-2">Módulo temporalmente no disponible</h2>
      <p class="text-amber-700 mb-4">
        El módulo de <strong>{{ nombreModulo() }}</strong> no se pudo cargar en este momento.
        Nuestro equipo ha sido notificado automáticamente.
      </p>
      <button (click)="reintentar()" class="px-4 py-2 bg-amber-600 text-white rounded hover:bg-amber-700">
        Reintentar carga
      </button>
    </div>
  `
})
export class MfeErrorBoundaryComponent {
  readonly nombreModulo = input.required<string>();
  
  reintentar(): void {
    window.location.reload();
  }
}
```

### Routing con Fallback en el Shell

```typescript
// shell/src/app/app.routes.ts
import { Routes } from "@angular/router";
import { loadRemoteModule } from "@angular-architects/native-federation";
import { MfeErrorBoundaryComponent } from "./core/components/mfe-error-boundary.component";

export const routes: Routes = [
  {
    path: "catalogo",
    loadChildren: () => loadRemoteModule("mfeCatalogo", "./routes")
      .then(m => m.catalogoRoutes)
      .catch((err) => {
        console.error("[MFE ERROR]: No se pudo cargar el MFE de Catálogo:", err);
        // Retornamos una ruta de fallback que renderiza el componente de error
        return [
          {
            path: "**",
            component: MfeErrorBoundaryComponent,
            data: { nombreModulo: "Catálogo de Productos" }
          }
        ] as Routes;
      })
  }
];
```

---

## 16.8 Server-Side Rendering (SSR) en Arquitecturas Federadas

Implementar SSR en una arquitectura de Micro Frontends es uno de los desafíos técnicos más avanzados del desarrollo frontend moderno. El problema fundamental es que el servidor Node.js del Shell necesita pre-renderizar componentes que viven físicamente en otros servidores remotos.

### Estrategias para SSR con Module Federation

#### Estrategia 1: SSR Solo en el Shell (Pragmática)
El Shell se renderiza en el servidor con su layout, menú y contenido estático. Los MFEs remotos se renderizan únicamente en el cliente (CSR) con `@defer` en placeholders:

```html
<!-- Shell Template: Los MFEs se cargan solo en el cliente -->
<header><!-- SSR: Renderizado en el servidor --></header>
<nav><!-- SSR: Renderizado en el servidor --></nav>

@defer (on idle) {
  <router-outlet></router-outlet> <!-- CSR: MFEs se montan aquí en el cliente -->
} @placeholder {
  <div class="skeleton-loader">Cargando módulo...</div>
} @error {
  <app-mfe-error-boundary nombreModulo="Contenido Principal"></app-mfe-error-boundary>
}

<footer><!-- SSR: Renderizado en el servidor --></footer>
```

#### Estrategia 2: SSR Completo con Node Federation (Avanzada)
Utilizar `@module-federation/node` para cargar los chunks remotos también en el servidor Node.js. Esto requiere que cada MFE publique un bundle compatible con Node.js además del bundle del navegador.

```typescript
// shell/server.ts (Configuración avanzada)
import { loadRemoteEntry } from "@module-federation/node";

async function prepararSSR() {
  // Pre-cargar los manifiestos de los remotos en el servidor Node.js
  await loadRemoteEntry({
    remoteEntry: "http://mfe-catalogo-server:4201/server/remoteEntry.js",
    remoteName: "mfeCatalogo"
  });
}
```

> **Nota de Producción**: La Estrategia 1 (SSR solo en Shell + CSR para MFEs) es utilizada por la mayoría de las empresas de Fortune 500 por su simplicidad operacional. La Estrategia 2 es para escenarios donde el SEO es absolutamente crítico en cada página de cada MFE (ej. un marketplace público como Amazon).

---

## 16.9 Monorepos con Nx y Micro Frontends

**Nx** (de Nrwl) es la herramienta estándar de facto para gestionar monorepos Angular a gran escala. Nx proporciona generadores específicos para crear arquitecturas de Micro Frontends con Module Federation de forma automatizada.

### Generación de un Workspace con Nx

```bash
# Crear un workspace Nx vacío
npx create-nx-workspace@latest empresa-platform --preset=angular-monorepo

# Generar el Shell (Host)
npx nx g @nx/angular:host shell --style=scss --standalone

# Generar un MFE Remoto conectado al Shell
npx nx g @nx/angular:remote mfe-catalogo --host=shell --style=scss --standalone
npx nx g @nx/angular:remote mfe-carrito --host=shell --style=scss --standalone

# Generar una librería compartida de UI
npx nx g @nx/angular:library shared/ui --standalone --buildable

# Generar una librería compartida de lógica de negocio
npx nx g @nx/angular:library shared/data-access --standalone --buildable
```

### Comandos Esenciales de Nx para MFEs

```bash
# Servir todo el sistema (Shell + todos los Remotos) simultáneamente
npx nx serve shell --devRemotes="mfe-catalogo,mfe-carrito"

# Compilar SOLO los proyectos afectados por los últimos cambios en Git
npx nx affected -t build

# Ejecutar tests SOLO de los proyectos afectados
npx nx affected -t test

# Visualizar el grafo de dependencias entre proyectos
npx nx graph
```

### Estructura de un Monorepo Nx con MFEs

```
empresa-platform/
├── apps/
│   ├── shell/                      # Host Application
│   │   ├── src/app/app.routes.ts   # Routing federado
│   │   ├── module-federation.config.ts
│   │   └── ...
│   ├── mfe-catalogo/               # Remote: Catálogo
│   │   ├── src/app/catalogo.routes.ts
│   │   ├── module-federation.config.ts
│   │   └── ...
│   └── mfe-carrito/                # Remote: Carrito
├── libs/
│   ├── shared/
│   │   ├── ui/                     # Componentes visuales compartidos
│   │   │   ├── src/lib/
│   │   │   │   ├── button/
│   │   │   │   ├── modal/
│   │   │   │   └── index.ts
│   │   ├── data-access/            # Servicios y estado compartido
│   │   │   ├── src/lib/
│   │   │   │   ├── auth.service.ts
│   │   │   │   ├── mfe-communication.service.ts
│   │   │   │   └── index.ts
│   │   └── models/                 # Interfaces y tipos compartidos
│   │       └── src/lib/
│   │           ├── producto.interface.ts
│   │           └── usuario.interface.ts
├── nx.json                         # Configuración global de Nx
├── tsconfig.base.json              # Paths de TypeScript compartidos
└── package.json
```

---

## 16.10 Testing de Micro Frontends

Testear Micro Frontends requiere una estrategia multi-capa que asegure tanto el funcionamiento aislado de cada MFE como la integración correcta en el sistema completo.

### Pirámide de Testing para MFEs

```
                    ┌─────────────┐
                    │   E2E Tests │  ← Playwright: flujo completo Shell + Remotos
                    │  (Pocos)    │
                   ┌┴─────────────┴┐
                   │ Integration   │  ← Contract Tests: verificar que los módulos
                   │  Tests        │    expuestos cumplen el contrato esperado
                  ┌┴───────────────┴┐
                  │   Unit Tests    │  ← Vitest: cada MFE de forma aislada
                  │   (Muchos)      │    sin cargar Federation
                  └─────────────────┘
```

### Contract Testing entre Shell y Remotos

El **Contract Testing** verifica que el módulo expuesto por un Remote cumple con la interfaz (el "contrato") que el Shell espera consumir. Esto previene roturas silenciosas cuando un equipo modifica la API pública de su MFE sin avisar:

```typescript
// contract-tests/catalogo.contract.spec.ts
import { describe, it, expect } from "vitest";

describe("Contrato del MFE Catálogo", () => {
  it("debe exportar 'catalogoRoutes' como un array de Routes", async () => {
    // Importamos directamente el módulo expuesto
    const modulo = await import("mfe-catalogo/routes");
    
    expect(modulo.catalogoRoutes).toBeDefined();
    expect(Array.isArray(modulo.catalogoRoutes)).toBe(true);
    expect(modulo.catalogoRoutes.length).toBeGreaterThan(0);
    
    // Verificamos que la primera ruta tenga la estructura esperada
    const primeraRuta = modulo.catalogoRoutes[0];
    expect(primeraRuta.path).toBeDefined();
  });
});
```

---

## 16.11 Caso de Estudio: Migración Progresiva de un Monolito a Micro Frontends

La migración de un monolito Angular existente a una arquitectura de Micro Frontends no debe hacerse de golpe. La estrategia recomendada es la **migración estranguladora (Strangler Fig Pattern)**, donde gradualmente extraes funcionalidades del monolito hacia MFEs independientes mientras el monolito sigue funcionando como Shell.

### Fases de Migración

```
┌─────────────────────────────────────────────────────────────────────┐
│ Fase 1: El monolito se convierte en Shell                           │
│ - Se instala Module/Native Federation en el monolito existente      │
│ - El monolito funciona exactamente igual que antes                  │
│ - No hay cambios funcionales visibles para el usuario               │
├─────────────────────────────────────────────────────────────────────┤
│ Fase 2: Extraer el primer MFE (el más independiente)                │
│ - Se identifica el dominio más desacoplado (ej. "Ayuda/FAQ")       │
│ - Se crea un nuevo proyecto Angular standalone                      │
│ - Se migra el código del monolito al nuevo MFE                      │
│ - El Shell carga el MFE vía Federation en la ruta correspondiente   │
│ - Se elimina el código migrado del monolito                         │
├─────────────────────────────────────────────────────────────────────┤
│ Fase 3: Extraer MFEs de dominio crítico (ej. "Catálogo", "Pagos") │
│ - Se repite el proceso de la Fase 2 para cada dominio               │
│ - Se establece la librería compartida de comunicación inter-MFE     │
│ - Cada equipo asume ownership de su MFE                             │
├─────────────────────────────────────────────────────────────────────┤
│ Fase 4: El monolito se reduce a un Shell puro                       │
│ - El Shell solo contiene: Layout global, Auth, Routing federado     │
│ - Toda la lógica de negocio vive en MFEs independientes             │
│ - Cada equipo despliega su MFE de forma autónoma                    │
└─────────────────────────────────────────────────────────────────────┘
```

### Criterios para Decidir los Límites de cada MFE

1. **Ownership organizacional**: ¿Quién es el equipo responsable de esta funcionalidad?
2. **Frecuencia de cambio**: ¿Esta funcionalidad cambia semanalmente o es estable durante meses?
3. **Acoplamiento de datos**: ¿Los datos de este dominio se consultan de forma independiente o siempre junto con otros?
4. **Autonomía de despliegue**: ¿Es posible desplegar esta parte sin afectar al resto?

Si una funcionalidad cruza los límites de dos dominios, pertenece a la librería compartida (`shared/`), no a un MFE.

---

## Resumen del Capítulo

* Los **Micro Frontends** descomponen una aplicación web monolítica en sub-aplicaciones autónomas que se integran en runtime, permitiendo que equipos independientes desarrollen, testeen y desplieguen de forma completamente desacoplada.
* **Webpack Module Federation** es la tecnología fundacional que permite compartir módulos JavaScript entre aplicaciones compiladas de forma independiente, utilizando el concepto de Host (Shell) y Remotes (MFEs).
* **Native Federation** representa la evolución moderna basada en Import Maps del estándar W3C, compatible con el builder Esbuild/Vite de Angular v17+ y eliminando la dependencia de Webpack.
* La **comunicación entre MFEs** debe implementarse mediante Custom Events del DOM (para baja complejidad), servicios compartidos singleton (para Angular-Angular) o estado global controlado en `window` (último recurso).
* Las **dependencias compartidas** (`@angular/core`, `rxjs`, etc.) deben configurarse como `singleton: true` para evitar duplicación de código en memoria y errores fatales de inyección de dependencias.
* La **resiliencia ante caídas** de MFEs remotos se implementa mediante fallbacks dinámicos en el routing del Shell que renderizan componentes de error amigables.
* **Nx** simplifica la gestión de monorepos con MFEs mediante generadores automáticos, el comando `affected` para builds/tests selectivos y la visualización del grafo de dependencias.
* La **migración progresiva** de un monolito a MFEs sigue el patrón Strangler Fig, extrayendo gradualmente dominios del monolito mientras este continúa funcionando como Shell.

Este capítulo concluye la teoría avanzada del libro. Has adquirido el conocimiento arquitectónico necesario para diseñar, implementar y operar aplicaciones Angular de cualquier escala, desde proyectos personales hasta plataformas empresariales distribuidas globalmente con decenas de equipos.
