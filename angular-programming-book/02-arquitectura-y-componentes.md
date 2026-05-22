# Capítulo 2: Arquitectura y Componentes Standalone

En el desarrollo de aplicaciones frontend modernas, la modularidad, el rendimiento y la facilidad de mantenimiento son pilares fundamentales. Angular, desde sus versiones v14/v17+, ha experimentado una de las revoluciones más importantes de su historia: la transición hacia la **Arquitectura Standalone (independiente)**.

Este capítulo profundiza en la estructura y el comportamiento de los componentes independientes, el nuevo modelo de compilación que elimina la sobrecarga conceptual de los `NgModule`, la anatomía del decorador `@Component`, el manejo reactivo de propiedades mediante **Signal-based Inputs y Outputs**, y el ciclo de vida de los componentes en aplicaciones empresariales de alto rendimiento.

---

## 2.1 ¿Qué es un Componente Standalone?

Tradicionalmente, en Angular, cada componente debía estar declarado obligatoriamente dentro de un `NgModule` (`@NgModule`). El módulo actuaba como un contenedor lógico que definía el contexto de compilación, indicando qué directivas, pipes y otros componentes estaban disponibles en las plantillas.

Aunque este enfoque organizaba la aplicación en bloques funcionales, introducía una gran fricción conceptual y técnica:
* **Sobrecarga de boilerplate**: Era necesario declarar e importar elementos en múltiples archivos solo para usar un simple componente o pipe.
* **Dificultad en Tree-Shaking**: Los módulos tendían a empaquetar código innecesario, dificultando la capacidad del compilador para eliminar código huérfano.
* **Curva de aprendizaje pronunciada**: Los desarrolladores noveles debían entender el sistema de módulos antes de poder renderizar un "Hola Mundo".

Un **Componente Standalone** elimina la necesidad de pertenecer a un `NgModule`. Es una **unidad de compilación autosuficiente**. Esto significa que el componente declara explícitamente sus propias dependencias directamente en sus metadatos.

### Comparación de Modelos de Compilación

```
Modelo Clásico (Basado en NgModules):
┌──────────────────────────────────────────────┐
│                  AppModule                   │
│  ┌─────────────────┐    ┌─────────────────┐  │
│  │   ComponentA    │    │   ComponentB    │  │
│  └────────┬────────┘    └────────┬────────┘  │
└───────────┼──────────────────────┼───────────┘
            ▼                      ▼
    (Dependen del contexto del módulo completo)

Modelo Moderno (Standalone):
┌───────────────────┐            ┌───────────────────┐
│    ComponentA     │            │    ComponentB     │
│  (standalone: true│            │  (standalone: true│
│   imports: [B])   │◄───────────┤   imports: [...]) │
└───────────────────┘            └───────────────────┘
   (Declara directamente sus dependencias a nivel individual)
```

Al marcar `standalone: true` en el decorador, le indicamos al compilador de Angular (Ivy) que procese este componente de forma aislada. Si el componente necesita utilizar una directiva (como la manipulación de clases) o invocar a otro componente, simplemente lo agrega a su lista de `imports`.

---

## 2.2 Anatomía del Decorador `@Component`

El decorador `@Component` es una función que añade metadatos a una clase de TypeScript, transformándola en un componente de Angular. Veamos la estructura moderna estándar de un componente de producción:

```typescript
import { Component, ChangeDetectionStrategy } from "@angular/core";
import { CommonModule } from "@angular/common";
import { TarjetaProductoComponent } from "../tarjeta-producto/tarjeta-producto.component";
import { FiltrarMonedaPipe } from "../../shared/pipes/filtrar-moneda.pipe";

@Component({
  selector: "app-lista-productos",
  standalone: true,
  imports: [
    CommonModule,
    TarjetaProductoComponent,
    FiltrarMonedaPipe
  ],
  templateUrl: "./lista-productos.component.html",
  styleUrl: "./lista-productos.component.css",
  changeDetection: ChangeDetectionStrategy.OnPush
})
export class ListaProductosComponent {
  // Lógica del componente
}
```

### Explicación de los Metadatos Clave

#### 1. `selector`
Define el nombre de la etiqueta HTML personalizada con la que se instanciará el componente en otras plantillas.
* *Convención*: Debe usar `kebab-case` y estar precedido por un prefijo identificador del proyecto (por ejemplo, `app-`, `ui-`, `auth-`) para evitar colisiones con futuras etiquetas estándar de HTML5.
* *Ejemplo*: `<app-lista-productos></app-lista-productos>`.

#### 2. `standalone`
Un valor booleano (`true`) que habilita la arquitectura independiente. A partir de Angular v19, este valor es `true` por defecto, pero explicitarlo sigue siendo una práctica excelente de legibilidad.

#### 3. `imports`
Aquí se especifican de forma explícita todos los elementos externos (componentes, directivas, pipes o módulos) que la plantilla de este componente consume. 
* *Ventaja*: Si tu plantilla no usa un componente determinado, no lo importas en este array. Esto permite un **tree-shaking extremadamente granular**. Tree-shaking (literalmente, "sacudir el árbol") es un proceso automático que elimina el código que no se usa, como podar las ramas secas de un árbol: si tu aplicación importa una biblioteca de 100 funciones pero solo usa 3, el tree-shaking elimina las otras 97 del archivo final que tus usuarios descargan. Resultado: archivos más pequeños y carga más rápida.

#### 4. `templateUrl` e Inline `template`
* `templateUrl`: Apunta a un archivo externo `.html` para definir la interfaz de usuario. Es la opción recomendada para componentes complejos o de gran envergadura (> 30 líneas de HTML).
* `template` (inline): Permite incrustar la plantilla directamente como un string literal en TypeScript. Es idóneo para componentes pequeños, reutilizables o puramente visuales (por ejemplo, un spinner o un botón personalizado).

```typescript
@Component({
  selector: "ui-badge",
  standalone: true,
  template: `
    <span class="inline-flex items-center px-2 py-1 text-xs font-semibold rounded bg-blue-100 text-blue-800">
      <span class="w-1.5 h-1.5 mr-1.5 rounded-full bg-blue-500"></span>
      {{ texto }}
    </span>
  `
})
export class BadgeComponent {
  texto = "En Stock";
}
```

#### 5. `styleUrl` e Inline `styles`
* `styleUrl` (introducido en Angular v17): Reemplaza al clásico array `styleUrls` y permite definir la ruta a un único archivo de hojas de estilo (CSS, SASS o LESS).
* `styles` (inline): Permite incrustar CSS directamente dentro de TypeScript mediante strings literales.
* *Encapsulamiento*: Por defecto, Angular empaqueta de forma hermética los estilos del componente mediante un sombreado emulado (usando atributos únicos como `_ngcontent-c123`), evitando que las reglas CSS del componente afecten de forma global a la aplicación y viceversa.

#### 6. `changeDetection`
Controla el algoritmo de detección de cambios de Angular.

> **📖 ¿Qué es la detección de cambios?** Cada vez que algo sucede en tu aplicación (un click, datos que llegan del servidor, un temporizador), Angular necesita **revisar** la pantalla para actualizar lo que el usuario ve. Este proceso de revisión se llama "detección de cambios". La estrategia `Default` revisa TODOS los componentes ante cualquier evento (como un guardia de seguridad paranoico que revisa todo el edificio cada vez que suena una alarma). La estrategia `OnPush` es más inteligente: solo revisa un componente cuando algo que le afecta directamente ha cambiado (como un guardia que solo revisa el piso donde sonó la alarma). El concepto de "cambio de referencia" mencionado abajo significa que Angular compara si el objeto es literalmente el mismo o uno nuevo, no si su contenido cambió.

* **`ChangeDetectionStrategy.OnPush` (Recomendado)**: Indica a Angular que solo debe ejecutar el ciclo de detección de cambios cuando las propiedades de entrada (`Input`) del componente cambien de referencia, cuando se emita un evento desde la plantilla, o cuando un Signal del que depende sufra modificaciones. Esto optimiza enormemente el rendimiento en comparación con la estrategia `Default`, la cual recorre todo el árbol de componentes ante cualquier evento asíncrono (como un `setTimeout` o un click en cualquier parte de la pantalla).

---

## 2.3 Propiedades de Entrada y Salida Modernas (Signal-based APIs)

Angular v17+ ha introducido las **Signal-based Inputs y Outputs**, que sustituyen la sintaxis tradicional de los decoradores `@Input()` y `@Output()` por APIs puramente reactivas que mejoran el rendimiento, el tipado y la legibilidad.

### Signal Inputs (`input`)

Los Inputs basados en Signals permiten que el componente reciba datos de su componente padre en forma de un Signal de sólo lectura. Esto facilita enormemente la computación derivada de propiedades sin necesidad de usar getters complejos o el hook `ngOnChanges`.

```typescript
import { Component, input, computed } from "@angular/core";

@Component({
  selector: "ui-tarjeta-usuario",
  standalone: true,
  template: `
    <div class="p-4 border rounded shadow bg-white">
      <h3 class="text-lg font-bold">{{ nombreCompleto() }}</h3>
      <p class="text-sm text-gray-600">ID de Cuenta: {{ id() }}</p>
    </div>
  `
})
export class TarjetaUsuarioComponent {
  // Input obligatorio (required)
  id = input.required<string>();

  // Input opcional con valor por defecto
  nombre = input<string>("Invitado");
  apellido = input<string>("");

  // Propiedad derivada automática y reactiva usando computed()
  nombreCompleto = computed(() => {
    return `${this.nombre()} ${this.apellido()}`.trim();
  });
}
```

#### Ventajas sobre el `@Input()` Clásico:
1. **Tipado Estricto**: `input.required<T>()` garantiza en tiempo de compilación que el componente padre debe proveer la propiedad, de lo contrario Angular arrojará un error.
2. **Reactividad Integrada**: Al ser un Signal, el componente reacciona inmediatamente al cambio de valor, facilitando la composición de estados mediante `computed()` y la sincronización con efectos secundarios (`effect()`).

### Outputs Modernos (`output`)

Para emitir eventos al componente padre, Angular proporciona la nueva API `output()`, que elimina la dependencia directa del módulo RxJS `EventEmitter` para casos sencillos de comunicación inter-componentes, ofreciendo un tipado mucho más limpio.

```typescript
import { Component, output } from "@angular/core";

@Component({
  selector: "ui-boton-accion",
  standalone: true,
  template: `
    <button (click)="emitirClick()" class="px-4 py-2 bg-blue-600 text-white rounded">
      Ejecutar Acción
    </button>
  `
})
export class BotonAccionComponent {
  // Declaración del evento de salida
  accionConfirmada = output<string>();

  emitirClick() {
    // Emisión tipada del evento hacia el padre
    this.accionConfirmada.emit("Click ejecutado por el usuario");
  }
}
```

El componente padre puede escuchar este evento en su plantilla mediante el enlace de eventos tradicional de Angular:
```html
<ui-boton-accion (accionConfirmada)="manejarAccion($event)"></ui-boton-accion>
```

---

## 2.4 Ciclo de Vida y Compilación de Componentes

Un componente de Angular pasa por una serie de fases desde su creación e instanciación hasta su eventual destrucción. Angular permite interceptar estos momentos clave a través de interfaces llamadas **Lifecycle Hooks (Ganchos del Ciclo de Vida)**.

### Fases del Ciclo de Vida (Orden Cronológico)

```
        Instanciación (Llamada al Constructor)
                       │
                       ▼
            ngOnChanges (Si existen @Inputs)
                       │
                       ▼
                   ngOnInit (Inicialización)
                       │
                       ▼
                  ngDoCheck (Detección a medida)
                       │
             ┌─────────┴─────────┐
             ▼                   ▼
    ngAfterContentInit  ngAfterViewInit (DOM listo)
             │                   │
             ▼                   ▼
    ngAfterContentChecked ngAfterViewChecked
             └─────────┬─────────┘
                       │
                       ▼
            ngOnDestroy (Limpieza y fugas)
```

1. **`constructor()`**: No es un hook de Angular, sino el constructor nativo de la clase de TypeScript. Su único propósito debe ser la **Inyección de Dependencias**. Jamás realices operaciones pesadas (como llamadas HTTP o lógica de negocio) en el constructor.
2. **`ngOnChanges(changes: SimpleChanges)`**: Se ejecuta inmediatamente después del constructor y cada vez que cambie el valor de un `@Input()` tradicional. Recibe un objeto que contiene el valor actual y el valor previo de cada propiedad modificada. *(Nota: Si usas la nueva API de `input()` basada en Signals, en su lugar utilizarás la composición reactiva de Signals, haciendo este hook prácticamente innecesario).*
3. **`ngOnInit()`**: Se ejecuta exactamente una vez después del primer `ngOnChanges`. Es el lugar idóneo para realizar **peticiones HTTP a APIs**, inicializaciones de lógica de negocio y configuraciones iniciales del componente.
4. **`ngDoCheck()`**: Ejecutado ante cada ciclo de detección de cambios. Úsalo con extrema cautela para implementar algoritmos personalizados de detección cuando las estrategias automáticas de Angular no sean suficientes.
5. **`ngAfterViewInit()`**: Se invoca una vez que las plantillas y vistas del propio componente (incluyendo sus componentes hijos) han sido renderizadas y están disponibles en el DOM. Es el lugar ideal para interactuar con librerías externas que necesiten manipular el DOM de forma directa (como Chart.js o D3.js).
6. **`ngOnDestroy()`**: Se ejecuta inmediatamente antes de que Angular desmonte y destruya el componente del DOM. Es un hook **crucial para la estabilidad de la aplicación**, ya que aquí debes realizar la limpieza de eventos globales, cancelar timers (`setInterval`, `setTimeout`) e invalidar suscripciones abiertas de Observables para prevenir **fugas de memoria (memory leaks)**.

### Implementación Práctica de Hooks en Producción

Veamos una clase completa con un escenario real de consumo de datos y limpieza de recursos:

```typescript
import { 
  Component, 
  OnInit, 
  AfterViewInit, 
  OnDestroy, 
  inject, 
  signal 
} from "@angular/core";
import { Subscription } from "rxjs";
import { UsuarioService } from "../../core/services/usuario.service";
import { Usuario } from "../../core/models/usuario.interface";

@Component({
  selector: "app-perfil-usuario",
  standalone: true,
  template: `
    @if (cargando()) {
      <p class="text-blue-500 animate-pulse">Cargando perfil...</p>
    } @else if (usuario()) {
      <div class="p-6 max-w-sm rounded bg-slate-50 border border-slate-200">
        <h2 class="text-xl font-bold">{{ usuario()?.nombre }}</h2>
        <p class="text-sm text-gray-500">{{ usuario()?.email }}</p>
      </div>
    }
  `
})
export class PerfilUsuarioComponent implements OnInit, AfterViewInit, OnDestroy {
  // 1. Inyección de dependencias moderna mediante la función inject()
  private readonly usuarioService = inject(UsuarioService);
  private subscription?: Subscription;

  // 2. Control de estado reactivo local con Signals
  // Los Signals son como pantallas de marcador: siempre muestran el valor actual
  // y se actualizan automáticamente cuando el valor cambia.
  usuario = signal<Usuario | null>(null);
  cargando = signal<boolean>(true);

  constructor() {
    console.log("1. Constructor: Inyección de dependencias completada.");
  }

  ngOnInit(): void {
    console.log("2. OnInit: Iniciando peticiones HTTP.");
    
    // Suscripción asíncrona segura.
    // Un Observable (de RxJS) es como suscribirte a un canal de noticias en vivo:
    // las noticias llegan una a una, a lo largo del tiempo. Es ideal para datos
    // que llegan del servidor de forma asíncrona.
    // IMPORTANTE: siempre debes "cancelar la suscripción" (unsubscribe) cuando
    // el componente se destruye, o las noticias seguirán llegando para siempre
    // (fuga de memoria). Los Signals, en cambio, se limpian automáticamente.
    this.subscription = this.usuarioService.obtenerPerfil().subscribe({
      next: (datos) => {
        this.usuario.set(datos);
        this.cargando.set(false);
      },
      error: () => this.cargando.set(false)
    });
  }

  ngAfterViewInit(): void {
    console.log("3. AfterViewInit: El DOM del componente ya está renderizado.");
  }

  ngOnDestroy(): void {
    console.log("4. OnDestroy: Limpiando recursos y previniendo fugas de memoria.");
    
    // Cancelar la suscripción antes de destruir el componente
    this.subscription?.unsubscribe();
  }
}
```

---

## 2.5 Creación de Componentes con Angular CLI

El ecosistema de Angular se beneficia enormemente del **Angular CLI**, una herramienta de terminal que automatiza la creación, el andamiaje y la compilación del código del proyecto.

Para generar un nuevo componente standalone de manera óptima y coherente con las mejores prácticas, abrimos la terminal en la raíz del proyecto y ejecutamos:

```bash
ng generate component modules/auth/components/login --change-detection=OnPush
```

O su versión abreviada:

```bash
ng g c modules/auth/components/login -c=OnPush
```

### Parámetros y Flags Esenciales para Producción

| Flag | Tipo | Propósito |
|------|------|-----------|
| `--change-detection=OnPush` | String | Configura automáticamente la estrategia `OnPush` para un rendimiento premium. |
| `--inline-template` (o `-t`) | Booleano | Define la plantilla HTML en el propio archivo TypeScript en lugar de crear un archivo `.html` adicional. |
| `--inline-style` (o `-s`) | Booleano | Define las reglas de estilo en el propio archivo TypeScript en lugar de un archivo `.css` adicional. |
| `--skip-tests` | Booleano | Omite la generación del archivo de pruebas unitarias `.spec.ts` (no recomendado para flujos de CI/CD estrictos). |

---

## 2.6 Buenas Prácticas de Organización y Cohesión

Para mantener una base de código escalable en grandes equipos y proyectos empresariales, se recomiendan las siguientes directrices arquitectónicas para el diseño de componentes:

### 1. El Patrón Smart Components vs. Dumb Components
Divide tus componentes en dos roles bien diferenciados:
* **Smart Components (Componentes Contenedores / Inteligentes)**:
  * Orquestan la lógica de negocio de la pantalla o sección.
  * Inyectan servicios y realizan peticiones de datos HTTP.
  * Escuchan los eventos emitidos por los componentes Dumb y actualizan el estado global.
  * Suelen ubicarse en carpetas de vistas o páginas (ej. `views/`, `pages/`).
* **Dumb Components (Componentes de Presentación / Tontos)**:
  * Son puramente visuales y reutilizables.
  * No inyectan servicios de negocio; son agnósticos a la fuente de datos.
  * Reciben datos únicamente a través de inputs (`input()`) y notifican interacciones de usuario exclusivamente mediante outputs (`output()`).
  * Suelen ubicarse en carpetas de layouts o interfaces comunes (ej. `shared/components/`, `ui/`).

### 2. Principio de Responsabilidad Única (SRP)
Si un componente supera las 200 líneas en su archivo TypeScript o tiene demasiadas responsabilidades visuales cruzadas, es un candidato claro para ser descompuesto en subcomponentes standalone más pequeños y especializados. Esto facilita exponencialmente la realización de pruebas unitarias rápidas e independientes.

---

## Resumen del Capítulo

* La **Arquitectura Standalone** elimina la necesidad de declarar componentes en `NgModule`, reduciendo el boilerplate y maximizando la eficiencia de las optimizaciones como el Tree-Shaking.
* El decorador `@Component` actúa como el núcleo del componente, donde configuramos el selector, sus dependencias explícitas en `imports`, y su motor de renderizado.
* La adopción obligatoria de **`ChangeDetectionStrategy.OnPush`** es el estándar de rendimiento en aplicaciones empresariales de gran escala, evitando ejecuciones de detección de cambios innecesarias.
* Las nuevas APIs basadas en Signals (`input()` y `output()`) proveen una reactividad nativa y fluida, tipado estricto en tiempo de compilación y una óptima interoperabilidad con las operaciones derivadas del estado.
* El ciclo de vida define con claridad dónde inyectar dependencias (`constructor`), dónde consumir APIs externas de forma segura (`ngOnInit`) y dónde prevenir memory leaks de manera explícita (`ngOnDestroy`).

En el próximo capítulo, exploraremos la potente sintaxis de las plantillas de Angular y dominaremos el nuevo **Control Flow sintáctico** (`@if`, `@for` y `@switch`) para modelar interfaces de usuario extremadamente dinámicas y de alto rendimiento.

---

← [Capítulo anterior](01-introduccion.md) | [Inicio](README.md) | [Capítulo siguiente →](03-templates-y-directivas.md)
