# Capítulo 7: Enrutamiento y Navegación

En las aplicaciones modernas de una sola página (SPA - Single Page Applications), la navegación entre diferentes vistas no debe implicar recargas totales de la página en el navegador. En su lugar, es el **enrutador (Router)** del framework quien se encarga de interceptar los cambios en la barra de direcciones URL del navegador, desmontar los componentes visuales de la vista actual y renderizar dinámicamente los nuevos componentes correspondientes de forma instantánea.

Angular cuenta con un sistema de enrutamiento robusto y maduro, adaptado en el Angular Renaissance a la **arquitectura funcional y standalone**.

En este capítulo, aprenderemos a configurar el enrutamiento moderno mediante `provideRouter`. Dominaremos las técnicas de **carga perezosa (Lazy Loading)** para mantener archivos ultra ligeros, aprenderemos a inyectar parámetros de ruta de forma directa en las propiedades de nuestros componentes, y blindaremos la seguridad de nuestras vistas implementando **Guards y Resolvers funcionales** de alto rendimiento.

---

## 7.1 Configuración Moderna del Enrutamiento (`provideRouter`)

En el Angular moderno sin módulos (`NgModule`), la configuración global del sistema de enrutamiento se declara directamente en el objeto de configuración global de la aplicación, el archivo `app.config.ts`, utilizando la función **`provideRouter()`**.

### 1. Definición del Archivo de Rutas: `app.routes.ts`

Definimos las reglas de mapeo entre URLs y componentes standalone de nuestra aplicación:

```typescript
import { Routes } from "@angular/router";
import { HomeComponent } from "./features/home/home.component";
import { PaginaNoEncontradaComponent } from "./shared/components/not-found.component";

export const routes: Routes = [
  // Ruta por defecto (Redirección inicial)
  { path: "", redirectTo: "home", pathMatch: "full" },
  
  // Ruta estática estándar
  { path: "home", component: HomeComponent },
  
  // Ruta comodín (Wildcard) para manejar errores 404
  { path: "**", component: PaginaNoEncontradaComponent }
];
```

---

### 2. Registrar el Proveedor en `app.config.ts`

Importamos y registramos las rutas utilizando `provideRouter` e incorporamos optimizaciones clave del enrutamiento moderno como la vinculación de parámetros a inputs:

```typescript
import { ApplicationConfig } from "@angular/core";
import { provideRouter, withComponentInputBinding, withInMemoryScrolling } from "@angular/router";
import { routes } from "./app.routes";

export const appConfig: ApplicationConfig = {
  providers: [
    // Proveedor del enrutador moderno
    provideRouter(
      routes,
      // 1. Mapea query y route parameters directamente a @Input() / input() de componentes
      withComponentInputBinding(),
      
      // 2. Restaura la posición del scroll al inicio de la página ante cada navegación
      withInMemoryScrolling({ scrollPositionRestoration: "enabled" })
    )
  ]
};
```

---

## 7.2 Carga Perezosa (Lazy Loading) de Componentes Standalone

Para garantizar que nuestra aplicación cargue en milisegundos en redes móviles o conexiones lentas, es vital implementar la **Carga Perezosa (Lazy Loading)**. Consiste en dividir nuestra base de código en paquetes más pequeños (chunks) que se descargan en el navegador únicamente cuando el usuario navega físicamente a la ruta asociada.

En la arquitectura standalone moderna, el lazy loading es ridículamente simple y se logra utilizando promesas nativas de importación dinámica (`import()`).

### 1. Cargar un Componente Standalone Individual: `loadComponent`

```typescript
export const routes: Routes = [
  {
    path: "ayuda",
    // El componente 'AyudaComponent' no se incluirá en el bundle inicial de la app
    loadComponent: () => import("./features/ayuda/ayuda.component").then(m => m.AyudaComponent)
  }
];
```

---

### 2. Cargar un Árbol Completo de Subrutas: `loadChildren`

Cuando tienes un módulo funcional completo (por ejemplo, el área administrativa `/admin` con múltiples pantallas), puedes encapsular todas las subrutas en un archivo independiente y cargarlas de forma perezosa en bloque:

```typescript
// Archivo de subrutas administrativas: features/admin/admin.routes.ts
import { Routes } from "@angular/router";

export const adminRoutes: Routes = [
  { 
    path: "", 
    loadComponent: () => import("./dashboard.component").then(m => m.AdminDashboardComponent) 
  },
  { 
    path: "usuarios", 
    loadComponent: () => import("./usuarios.component").then(m => m.AdminUsuariosComponent) 
  }
];

// Archivo principal: app.routes.ts
export const routes: Routes = [
  {
    path: "admin",
    // Carga de forma perezosa todo el árbol de rutas administrativas
    loadChildren: () => import("./features/admin/admin.routes").then(m => m.adminRoutes)
  }
];
```

---

## 7.3 Extracción de Parámetros Directa a Component Inputs

Tradicionalmente, para leer un parámetro dinámico de la URL (como el `id` en `/productos/:id`), debías inyectar el servicio `ActivatedRoute` en el constructor de tu componente y suscribirte a un observable:

```typescript
// Enfoque clásico (Vervoso e incómodo)
this.route.paramMap.subscribe(params => {
  this.productoId = params.get("id");
});
```

En Angular moderno, gracias a la bandera **`withComponentInputBinding()`** registrada en `app.config.ts`, los parámetros se extraen e inyectan **directamente como Inputs** en el componente de manera transparente.

### Implementación Práctica:
Definimos la ruta dinámica en `app.routes.ts`:
```typescript
{
  path: "productos/:id",
  loadComponent: () => import("./features/detalle/detalle.component").then(m => m.DetalleComponent)
}
```

Capturamos el parámetro dinámico `id` utilizando la nueva API `input()` basada en Signals:

```typescript
import { Component, input, computed } from "@angular/core";

@Component({
  selector: "app-detalle-producto",
  standalone: true,
  template: `
    <div class="p-6">
      <h2>Visualizando Producto #{{ id() }}</h2>
      <p>Los detalles del registro se consultan para el identificador recibido.</p>
    </div>
  `
})
export class DetalleComponent {
  // El valor del input coincide exactamente con el nombre ':id' definido en la ruta.
  // Angular realiza la vinculación de tipos automáticamente.
  readonly id = input.required<string>();
}
```

---

## 7.4 Seguridad y Flujos con Guards de Ruta Funcionales

Los **Guards** son mecanismos de protección que determinan si una navegación a una ruta determinada se puede permitir (`true`) o rechazar (`false` o redirección).

En el Angular moderno, los guards basados en clases están deprecados. En su lugar, se utilizan **Guards Funcionales**. Son funciones puras que pueden usar la inyección funcional a través de `inject()` de forma elegante para resolver la lógica de seguridad.

### Creación de un Guard de Autenticación Funcional: `authGuard`

Construiremos un guard funcional que bloquee el acceso a usuarios no autenticados y los redirija de manera segura a la pantalla de `/login`.

```typescript
import { inject } from "@angular/core";
import { CanActivateFn, Router } from "@angular/router";
import { AuthService } from "../services/auth.service";

export const authGuard: CanActivateFn = (route, state) => {
  // Inyección funcional de dependencias dentro de la función pura
  const authService = inject(AuthService);
  const router = inject(Router);

  if (authService.usuarioLogueado()) {
    return true; // Acceso permitido
  }

  console.warn(`[GUARD WARNING]: Intento de acceso no autorizado a: ${state.url}`);
  
  // Si no está autenticado, retornamos una redirección (UrlTree) de forma segura
  return router.parseUrl("/login");
};
```

#### Aplicación en el Archivo de Rutas:
```typescript
import { authGuard } from "./core/guards/auth.guard";

export const routes: Routes = [
  {
    path: "admin",
    loadChildren: () => import("./features/admin/admin.routes").then(m => m.adminRoutes),
    // Protege la ruta raíz y todas las subrutas hijas de forma recursiva
    canActivate: [authGuard] 
  }
];
```

---

## 7.5 Pre-carga de Datos con Resolvers Funcionales

Navegar a una pantalla y encontrarse con múltiples spinners mientras se cargan los datos puede dañar seriamente la experiencia de usuario. Los **Resolvers** permiten pre-cargar los datos necesarios en segundo plano *antes* de que el componente sea renderizado. Si la llamada asíncrona falla o tarda, la navegación se interrumpe, evitando pantallas a medias.

Al igual que los guards, los resolvers modernos son **funcionales**.

### Creación de un Resolver Funcional de Productos

```typescript
import { inject } from "@angular/core";
import { ResolveFn } from "@angular/router";
import { ProductoService } from "../services/producto.service";
import { Producto } from "../models/producto.interface";

export const productoResolver: ResolveFn<Producto> = (route) => {
  const productoService = inject(ProductoService);
  
  // Extraemos el parámetro 'id' de la URL solicitada
  const id = route.paramMap.get("id");

  if (!id) {
    throw new Error("Identificador de producto no suministrado en la ruta.");
  }

  // Retorna el Observable asíncrono. El router se suscribirá internamente en segundo plano
  // y completará la navegación únicamente cuando los datos estén listos.
  return productoService.obtenerDetalle(id);
};
```

---

#### Uso del Resolver en la Configuración de Rutas:
Definimos la propiedad `resolve` en el objeto de configuración de la ruta:

```typescript
export const routes: Routes = [
  {
    path: "productos/:id",
    loadComponent: () => import("./features/detalle/detalle.component").then(m => m.DetalleComponent),
    resolve: {
      // Clave para mapear los datos resueltos
      productoCargado: productoResolver 
    }
  }
];
```

#### Capturar los Datos en el Componente:
Debido a la bandera `withComponentInputBinding()`, los datos resueltos en `productoCargado` se inyectan directamente en el componente como un **Input** del mismo nombre de forma transparente:

```typescript
import { Component, input } from "@angular/core";
import { Producto } from "../../core/models/producto.interface";

@Component({
  selector: "app-detalle",
  standalone: true,
  template: `
    <div class="p-6">
      <h1 class="text-2xl font-bold">{{ productoCargado().nombre }}</h1>
      <p class="text-green-600 font-semibold">{{ productoCargado().precio }} €</p>
      <p class="mt-4">{{ productoCargado().descripcion }}</p>
    </div>
  `
})
export class DetalleComponent {
  // Captura automática de los datos pre-cargados por el resolver
  readonly productoCargado = input.required<Producto>();
}
```

---

## Resumen del Capítulo

* En Angular moderno, la navegación se configura utilizando **`provideRouter()`** en `app.config.ts`, permitiendo inicializaciones sin boilerplate.
* La **carga perezosa (Lazy Loading)** con `loadComponent` y `loadChildren` es una práctica mandatoria en producción para dividir el bundle de la aplicación en chunks optimizados.
* La función **`withComponentInputBinding()`** simplifica radicalmente el acoplamiento de datos del enrutador con los componentes, mapeando route params y resolvers directamente a inputs.
* Los **Guards Funcionales** representan un enfoque de programación pura y funcional que facilita la protección de rutas inyectando servicios de sesión mediante la función `inject()`.
* Los **Resolvers Funcionales** pre-cargan flujos de información en segundo plano, mejorando drásticamente la experiencia de usuario final al evitar transiciones visuales bruscas y spinners innecesarios.

En el próximo capítulo, aprenderemos cómo conectar nuestra aplicación a servidores remotos de base de datos dominando el cliente de comunicación asíncrona **`HttpClient`** y los operadores clave de **RxJS**.

---

← [Capítulo anterior](06-formularios-y-validaciones.md) | [Inicio](README.md) | [Capítulo siguiente →](08-comunicacion-asincrona-http.md)
