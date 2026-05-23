# Capítulo 5: Servicios e Inyección de Dependencias

Para construir aplicaciones robustas, escalables y mantenibles en el tiempo, es imprescindible seguir el principio de **Responsabilidad Única (Single Responsibility Principle)**. Los componentes deben centrarse exclusivamente en la lógica de presentación y el renderizado visual del DOM. La lógica de negocio pesada, la gestión de estado y el consumo de APIs remotas deben delegarse a unidades de software aisladas y altamente reutilizables llamadas **Servicios**.

Angular proporciona uno de los sistemas de **Inyección de Dependencias (DI - Dependency Injection)** más sofisticados, maduros y potentes de toda la ingeniería de software. 

En este capítulo, exploraremos en profundidad la arquitectura del motor de DI en el Angular moderno. Aprenderemos a crear servicios modulares utilizando la nueva función **`inject()`**, entenderemos la jerarquía interna de inyectores que rige a la aplicación y dominaremos técnicas de provisión avanzada mediante `InjectionToken` para proyectos empresariales.

---

## 5.1 El Patrón de Inyección de Dependencias (DI) en Angular

La **Inyección de Dependencias** es un patrón de diseño de software en el que una clase (como un componente) no crea ni instancia manualmente las dependencias externas que necesita para funcionar (por ejemplo, un servicio HTTP o de autenticación). En su lugar, es el **inyector central de Angular** quien se encarga de crear, instanciar y proveer estas dependencias de manera automática en tiempo de ejecución.

```
┌───────────────────────────────────────────────────────────────┐
│                    Inyector de Angular (DI)                   │
│   (Busca dependencias, administra instancias singleton)       │
└──────────────────────────────┬────────────────────────────────┘
                               │
            Provee la instancia│ (Resuelve dependencias)
                               ▼
┌───────────────────────────────────────────────────────────────┐
│                      Consumidor (Componente)                  │
│   (Solicita dependencias sin saber cómo se construyen)        │
└───────────────────────────────────────────────────────────────┘
```

### Ventajas Nucleares de usar Inyección de Dependencias:
* **Bajo Acoplamiento**: Los componentes son agnósticos a los detalles de implementación de los servicios. Si la forma en que se obtienen los datos cambia (por ejemplo, de REST a GraphQL), el componente no necesita sufrir ninguna modificación.
* **Fácil de Testear (Mocking)**: Durante las pruebas unitarias, podemos inyectar fácilmente implementaciones simuladas o duplicados de prueba (mock services) en lugar de conectarnos a servidores reales.
* **Gestión de Memoria y Ciclos de Vida**: El motor de DI gestiona si un servicio debe ser una única instancia compartida por toda la aplicación (**Singleton**) o si debe crearse una nueva instancia específica para un componente y destruirse automáticamente con él.

---

## 5.2 Creación de Servicios con `@Injectable`

Un servicio en Angular es una clase de TypeScript estándar decorada con el metadato **`@Injectable`**, el cual le indica al compilador que la clase puede ser administrada por el contenedor de inyección de dependencias de Angular.

Generamos un servicio estándar con el CLI:
```bash
ng generate service core/services/auth --standalone
```

Veamos la anatomía de un servicio moderno de producción:

```typescript
import { Injectable, signal } from "@angular/core";

@Injectable({
  providedIn: "root" // Configura el servicio como un Singleton global
})
export class AuthService {
  // Estado de sesión reactivo interno usando Signals
  private readonly _usuarioLogueado = signal<boolean>(false);
  readonly usuarioLogueado = this._usuarioLogueado.asReadonly();

  iniciarSesion(token: string): void {
    localStorage.setItem("authToken", token);
    this._usuarioLogueado.set(true);
  }

  cerrarSesion(): void {
    localStorage.removeItem("authToken");
    this._usuarioLogueado.set(false);
  }
}
```

### Explicación del Metadato `providedIn: 'root'`

La propiedad `providedIn: 'root'` le indica a Angular que este servicio debe ser registrado en el **inyector raíz (Root Injector)** de la aplicación. 
* **Instancia Única (Singleton)**: Toda la aplicación compartirá la misma y única instancia en memoria del servicio.
* **Tree-Shaking Automático**: Si el compilador detecta que tu aplicación nunca importa ni consume el `AuthService` en ningún componente, el compilador **eliminará el servicio por completo** del bundle de producción final, reduciendo el tamaño del archivo descargado por los usuarios.

---

## 5.3 Constructor-based DI frente a la Moderna Función `inject()`

Históricamente, la única manera de inyectar dependencias en Angular era declarándolas como parámetros privados dentro del `constructor` de la clase. Angular moderno (v16+) ha revolucionado esto introduciendo la función **`inject()`**.

### Comparación Sintáctica

#### 1. Enfoque Clásico (Constructor):
```typescript
import { Component } from "@angular/core";
import { Router } from "@angular/router";
import { AuthService } from "../../core/services/auth.service";

@Component({
  selector: "app-dashboard",
  standalone: true
})
export class DashboardComponent {
  // Inyección clásica a través del constructor
  constructor(
    private router: Router,
    private authService: AuthService
  ) {}

  salir() {
    this.authService.cerrarSesion();
    this.router.navigate(["/login"]);
  }
}
```

#### 2. Enfoque Moderno (Función `inject()`):
```typescript
import { Component, inject } from "@angular/core";
import { Router } from "@angular/router";
import { AuthService } from "../../core/services/auth.service";

@Component({
  selector: "app-dashboard",
  standalone: true
})
export class DashboardComponent {
  // Inyección funcional moderna
  private readonly router = inject(Router);
  private readonly authService = inject(AuthService);

  salir() {
    this.authService.cerrarSesion();
    this.router.navigate(["/login"]);
  }
}
```

---

### ¿Por qué `inject()` es infinitamente superior?

La función `inject()` no es solo azúcar sintáctico; resuelve serios límites arquitectónicos del desarrollo en TypeScript:

1. **Tipado Fuertemente Inferido**: Ya no necesitas usar modificadores como `private readonly` y escribir repetitivamente el tipo en los parámetros del constructor. El tipo se infiere de forma óptima a partir de la firma de retorno de `inject()`.
2. **Herencia Limpia de Clases**: En el enfoque clásico, si creas una clase base abstracta y la extiendes en múltiples clases hijas, estabas obligado a re-declarar todas las dependencias del constructor de la base en cada una de las clases hijas mediante `super(dep1, dep2...)`. Con `inject()`, la clase base inyecta sus dependencias directamente y las hijas las heredan de forma transparente y sin boilerplate.
3. **Inyección en Funciones y Componentes Funcionales**: Permite inyectar dependencias dentro de funciones utilitarias externas o **Guards de rutas funcionales** sin necesidad de envolverlos en clases o inyectores complejos.
4. **Contexto de Inicialización Seguro**: `inject()` debe invocarse durante la fase de instanciación de la clase (en la declaración de propiedades o dentro del `constructor`). Esto garantiza que el grafo de dependencias esté completamente resuelto y sea inmutable durante la ejecución de los hooks de ciclo de vida.

---

## 5.4 Jerarquía de Inyectores en Angular

La Inyección de Dependencias en Angular no es plana; se organiza como un **árbol jerárquico de inyectores** que se asemeja al propio árbol de componentes del DOM. Esto significa que si un componente solicita una dependencia, Angular la busca en su inyector local. Si no la encuentra, escala hacia arriba en la jerarquía hasta llegar al inyector raíz.

> [!NOTE]
> ### 🏢 La Analogía del Conserje del Hotel y los Suministros (Jerarquía de Inyectores)
> 
> Para comprender cómo Angular gestiona la resolución y el aislamiento de tus servicios a través del árbol, imagina que te hospedas en un gigantesco hotel de lujo de 5 estrellas:
> 
> - **El Minibar de tu Habitación (Local Component Injector / `ElementInjector`)**: Es una provisión privada y exclusiva de tu suite. Si tienes sed, abres la nevera de tu habitación y tomas una bebida. Esta bebida solo la tienes tú, y cuando haces el checkout (**el componente se desmonta y destruye**), el personal vacía y limpia la nevera, liberando la memoria.
> - **El Casillero de Housekeeping del Piso (Route Injector / `EnvironmentInjector`)**: Es un almacén intermedio ubicado en el pasillo de tu planta. Guarda mantas extras y toallas para las habitaciones de esa planta. Solo está disponible para los huéspedes que se alojan en ese piso específico (**la ruta activa y sus sub-rutas**).
> - **El Almacén Central del Vestíbulo (Root Injector / `EnvironmentInjector` Global)**: Es el almacén principal del hotel en el sótano. Tiene todos los suministros permanentes, maletas, repuestos y amenities globales. Es permanente, existe durante toda la vida del hotel (**Singleton global**) y todos los huéspedes de todos los pisos tienen acceso a él.
> 
> **¿Cómo resuelve Angular tu llamada a un servicio?**
> Si solicitas una botella de agua mineral (`inject(WaterService)`):
> 
> 1. Angular busca primero en tu minibar privado (**ElementInjector**). Si la encuentra, te la entrega y listo.
> 2. Si el minibar está vacío, el conserje sale al pasillo y revisa el casillero de la planta (**Route Injector**).
> 3. Si tampoco está en la planta, el conserje baja en ascensor hasta el almacén central del vestíbulo (**Root Injector**).
> 4. Si el almacén central del vestíbulo tiene la botella, te la sube a la habitación. Si tampoco existe en el almacén central, el conserje regresa con las manos vacías y te dice: *"Lo siento, no ofrecemos ese servicio en este hotel"* (**`NullInjectorError`**).

Existen dos tipos principales de inyectores en el entorno moderno:

```
                  ┌────────────────────────────────────────┐
                  │          EnvironmentInjector           │
                  │   - Root Injector (Singleton Global)    │
                  │   - Route Injector (Lazy Loaded Routes)│
                  └───────────────────┬────────────────────┘
                                      │ (Escalada de búsqueda)
                                      ▼
                  ┌────────────────────────────────────────┐
                  │            ElementInjector             │
                  │   - Component Injector (Local)         │
                  │   - Child Component Injector           │
                  └────────────────────────────────────────┘
```

### 1. `EnvironmentInjector` (Inyector de Entorno)
Gestiona servicios globales o específicos de un área configurada.
* **Root Injector**: Creado al inicializar la aplicación. Aloja los servicios globales (`providedIn: 'root'`) y configuraciones compartidas.
* **Route Injector**: Creado cuando utilizas `provideRouter` o cargas perezosamente rutas. Permite modularizar proveedores para que solo existan cuando el usuario navegue a cierta sección de la aplicación.

### 2. `ElementInjector` (Inyector de Elementos DOM)
Creado en cada componente individual de forma automática.
* Se alimenta del array `providers` o `viewProviders` definidos en el decorador `@Component`.
* Las dependencias provistas aquí no son globales; **tienen el mismo ciclo de vida que el componente**. Si el componente se destruye, todas las instancias de los servicios declarados en su inyector local se destruyen también de la memoria del navegador.

---

## 5.5 Patrones de Provisión y Ámbitos de Servicios (Scopes)

Podemos controlar el ciclo de vida, la visibilidad y el número de instancias de nuestros servicios mediante tres niveles de provisión:

### 1. Provisión Global (Root)
El estándar de facto para la mayoría de los servicios (APIs, logs, configuraciones globales).
```typescript
@Injectable({
  providedIn: 'root'
})
export class LoggerGlobalService {}
```

---

### 2. Provisión a Nivel de Ruta (Route Scope)
Permite proveer una instancia única de un servicio que sea compartida **únicamente por una ruta específica y sus componentes hijos**. Se configura en el archivo `app.routes.ts`:

```typescript
import { Routes } from "@angular/router";
import { ReporteService } from "./core/services/reporte.service";

export const routes: Routes = [
  {
    path: "dashboard/reportes",
    loadComponent: () => import("./features/reportes/reportes.component"),
    // El servicio estará disponible únicamente dentro de esta ruta y sus subrutas
    providers: [ReporteService] 
  }
];
```

---

### 3. Provisión a Nivel de Componente (Component Scope)
Crea una instancia aislada y fresca del servicio para cada componente que lo consuma en la pantalla. Es perfecto para componentes altamente dinámicos y reutilizables que mantengan un estado interno único (como un modal interactivo, un editor de texto o un widget financiero).

```typescript
import { Component, inject } from "@angular/core";
import { EstadoWidgetService } from "./estado-widget.service";

@Component({
  selector: "ui-widget-financiero",
  standalone: true,
  template: `
    <div class="p-4 border rounded">
      <h3>Datos de Mercado</h3>
      <!-- Contenido visual -->
    </div>
  `,
  // Cada widget en la pantalla creará su propia instancia de EstadoWidgetService
  providers: [EstadoWidgetService] 
})
export class WidgetFinancieroComponent {
  // Inyección de la instancia local específica de este widget
  private readonly estadoService = inject(EstadoWidgetService); 
}
```

---

## 5.6 Uso de `InjectionToken` y Proveedores Personalizados

En aplicaciones avanzadas, muchas veces necesitamos inyectar valores que no son clases de TypeScript (como strings, configuraciones de variables de entorno, APIs de terceros o constantes de configuración). Para lograr esto de forma tipada y segura, Angular proporciona la API **`InjectionToken`**.

### 1. Creación de un `InjectionToken` de Configuración

Definimos una interfaz para nuestra configuración y creamos el token:

```typescript
import { InjectionToken } from "@angular/core";

export interface AppConfig {
  apiUrl: string;
  version: string;
  timeoutMs: number;
}

// Creamos el token de inyección
export const APP_CONFIG = new InjectionToken<AppConfig>("app.environment.config");
```

---

### 2. Registrar el Proveedor en `app.config.ts`

Utilizamos la propiedad **`useValue`** para asociar el token con un objeto físico de configuración:

```typescript
import { ApplicationConfig } from "@angular/core";
import { APP_CONFIG, AppConfig } from "./app-config.token";

const configuracionProduccion: AppConfig = {
  apiUrl: "https://api.empresa.com/v1",
  version: "2.4.0-prod",
  timeoutMs: 5000
};

export const appConfig: ApplicationConfig = {
  providers: [
    // Registramos el token personalizado
    { provide: APP_CONFIG, useValue: configuracionProduccion }
  ]
};
```

---

### 3. Inyectar el Token en Servicios o Componentes

Para consumir el token utilizando la función `inject()`, simplemente lo pasamos como argumento:

```typescript
import { Injectable, inject } from "@angular/core";
import { HttpClient } from "@angular/common/http";
import { APP_CONFIG } from "./app-config.token";

@Injectable({
  providedIn: "root"
})
export class ApiClientService {
  private readonly http = inject(HttpClient);
  // Inyectamos el token de configuración tipado
  private readonly config = inject(APP_CONFIG);

  obtenerReporte() {
    console.log(`Conectándose a: ${this.config.apiUrl} (v${this.config.version})`);
    return this.http.get(`${this.config.apiUrl}/datos-financieros`);
  }
}
```

---

## Resumen del Capítulo

* El **patrón de Inyección de Dependencias** promueve el desacoplamiento al delegar la creación y el ciclo de vida de los servicios al inyector de Angular.
* La moderna función **`inject()`** es el estándar del Angular Renaissance, eliminando la verbosidad del constructor clásico y permitiendo inyecciones limpias en funciones, guards y herencias.
* La jerarquía interna de inyectores está estructurada en **`EnvironmentInjector`** (servicios globales y de ruta) y **`ElementInjector`** (servicios locales atados al ciclo de vida del componente).
* La propiedad `providedIn: 'root'` crea servicios Singleton inteligentes capaces de realizar **Tree-Shaking automático** a nivel de compilación.
* Los **`InjectionToken`** permiten inyectar constantes, APIs externas y objetos de configuración de manera robusta, tipada y segura.

En el próximo capítulo, aprenderemos a dominar la construcción de interfaces interactivas complejas construyendo y validando **Formularios Reactivos** de alto rendimiento en Angular moderno.

---

← [Capítulo anterior](04-signals.md) | [Inicio](README.md) | [Capítulo siguiente →](06-formularios-y-validaciones.md)
