# Capítulo 3: Templates, Control Flow Sintáctico y Directivas Avanzadas

> "Una plantilla de Angular no es un simple archivo HTML con variables. Es un lenguaje de dominio específico (DSL) compilado a instrucciones de renderizado optimizadas que el framework ejecuta contra un árbol de vistas virtual."

La plantilla es el contrato visual entre la lógica de negocio del componente (escrita en TypeScript) y la interfaz de usuario que experimenta el usuario final (renderizada como DOM nativo). A diferencia de frameworks como React, donde la vista se construye con funciones JavaScript puras (JSX), Angular utiliza un **compilador de plantillas ahead-of-time (AOT)** que transforma las declaraciones HTML enriquecidas en funciones de bajo nivel que manipulan el DOM de forma directa, eliminando la necesidad de un Virtual DOM intermedio.

En las versiones modernas de Angular (v17+), el compilador de plantillas Ivy ha sido completamente rediseñado para incorporar el **Control Flow Sintáctico nativo** (`@if`, `@for`, `@switch`), una característica que no solo simplifica la legibilidad del código sino que revoluciona el rendimiento de renderizado al eliminar el overhead de las directivas estructurales clásicas. Los benchmarks internos del equipo de Angular demuestran mejoras de hasta un **90% en el rendering de listas** y un **50% en el rendering condicional** comparado con `*ngIf` y `*ngFor`.

En este capítulo, dominaremos la sintaxis completa de plantillas, exploraremos cómo el compilador Ivy transforma internamente el control flow en instrucciones optimizadas, construiremos directivas personalizadas de nivel profesional (incluyendo la nueva API `hostDirectives`), aprenderemos a proyectar contenido dinámico con `ng-content` y `@ContentChild`, y blindaremos nuestras plantillas contra ataques XSS con `DomSanitizer`.

---

## 3.1 Sintaxis de Plantillas y Enlace de Datos (Bindings)

Angular utiliza una sintaxis enriquecida basada en HTML que establece conexiones reactivas entre la lógica del componente y la vista. Existen cuatro mecanismos fundamentales de enlace de datos, cada uno con semántica, dirección de flujo y comportamiento de detección de cambios diferentes.

```
                  ┌─────────────────────────────────────────┐
                  │          Lógica en TypeScript           │
                  └──────────┬───────────────────▲──────────┘
                             │                   │
               Interpolación │ (1)               │ Event Binding (3)
       Property Binding (2)  │                   │ (click)="event()"
           [prop]="value"    ▼                   │
                  ┌──────────────────────────────┴──────────┐
                  │             Plantilla HTML              │
                  │   ┌─────────────────────────────────┐   │
                  │   │     Two-way Binding (4)         │   │
                  │   │     [(ngModel)]="valor"         │   │
                  │   └─────────────────────────────────┘   │
                  └─────────────────────────────────────────┘
```

### 1. Interpolación (`{{ }}`)

La interpolación permite incrustar valores dinámicos de la clase TypeScript directamente en el texto del HTML. Angular evalúa la expresión, la convierte a string y la inyecta de forma segura en el DOM.

**Detalle interno del compilador**: Cuando Ivy encuentra una expresión de interpolación como `{{ usuario().nombre }}`, la transforma en una instrucción `ɵɵtextInterpolate1()` que compara el valor actual con el valor previo almacenado en la vista (LView). Si el valor ha cambiado, actualiza el `textContent` del nodo DOM directamente. Si no ha cambiado, la instrucción no realiza ninguna operación DOM, evitando repintadas costosas.

```html
<!-- Interpolación con expresiones complejas -->
<p>Bienvenido de nuevo, <strong>{{ usuario().nombre }}</strong></p>
<p>Saldo disponible: {{ saldo() * 1.16 | number:'1.2-2' }} €</p>
<p>Último acceso: {{ ultimoAcceso() | date:'dd/MM/yyyy HH:mm' }}</p>

<!-- Interpolación con operador ternario (evitar lógica compleja aquí) -->
<span class="estado">{{ usuario().activo ? 'En línea' : 'Desconectado' }}</span>
```

> **Advertencia de rendimiento**: Jamás coloques llamadas a métodos costosos (como filtros, sorts o peticiones) directamente en interpolaciones. Angular re-evalúa CADA expresión de interpolación en CADA ciclo de detección de cambios. Si tu componente usa `Default` strategy, eso significa que `ordenarListaCompleja()` se ejecutará ante cualquier click en cualquier parte de la pantalla. Usa `computed()` o pipes puros.

---

### 2. Enlace de Propiedades (Property Binding - `[ ]`)

Establece de manera unidireccional (TypeScript → HTML) el valor de una **propiedad del DOM** o de un `input()` de un componente hijo.

#### La Diferencia Crítica: Atributo HTML vs. Propiedad DOM

Esta es una distinción que el 90% de los desarrolladores Angular ignoran, pero que es fundamental para entender por qué ciertos bindings no funcionan como se espera:

* Un **atributo HTML** es el valor inicial declarado en el markup estático (`<input value="hola">`). Los atributos son **estáticos** y solo existen en el HTML serializado.
* Una **propiedad DOM** es el valor dinámico gestionado por el motor de renderizado del navegador (`inputElement.value`). Las propiedades pueden cambiar en runtime.

Cuando Angular ejecuta `[disabled]="formularioInvalido()"`, está manipulando la **propiedad DOM** `disabled` del elemento, no el **atributo HTML** `disabled`. Esto es importante porque algunos atributos (como `colspan`, `aria-*` o `data-*`) no tienen una propiedad DOM equivalente. Para enlazar atributos directamente, usamos el prefijo `attr.`:

```html
<!-- Property Binding: manipula la PROPIEDAD DOM -->
<button [disabled]="formularioInvalido()">Enviar</button>
<img [src]="producto().imagenUrl" [alt]="producto().nombre">

<!-- Attribute Binding: manipula el ATRIBUTO HTML directamente -->
<td [attr.colspan]="columnasVisibles()">Celda expandida</td>
<div [attr.role]="esModal() ? 'dialog' : 'region'" [attr.aria-hidden]="!visible()">...</div>

<!-- Class Binding: activa/desactiva clases CSS individuales -->
<div [class.active]="pestanaActiva() === 'inicio'"
     [class.disabled]="deshabilitado()"
     [class.error]="formulario.invalid">
  Contenido dinámico
</div>

<!-- Style Binding: establece propiedades CSS individuales -->
<div [style.width.px]="progreso() * 3"
     [style.background-color]="progreso() > 80 ? '#22c55e' : '#3b82f6'">
  {{ progreso() }}%
</div>
```

---

### 3. Enlace de Eventos (Event Binding - `( )`)

Permite escuchar interacciones del usuario y ejecutar métodos del componente. La variable especial `$event` contiene el objeto de evento nativo del navegador o el valor emitido por un output del componente hijo.

```html
<!-- Eventos nativos del navegador -->
<button (click)="guardarCambios($event)">Guardar</button>
<input (keyup.enter)="buscarProducto($event)" placeholder="Enter para buscar">
<div (mouseenter)="mostrarTooltip()" (mouseleave)="ocultarTooltip()">Hover me</div>

<!-- Pseudo-eventos de Angular para teclado (key events con filtro) -->
<input (keydown.control.s)="guardarAtajo($event)" placeholder="Ctrl+S para guardar">
<input (keydown.escape)="cerrarModal()">

<!-- Eventos de componentes hijo (outputs) -->
<app-tarjeta-producto (comprar)="agregarAlCarrito($event)"></app-tarjeta-producto>

<!-- Prevenir comportamiento por defecto del navegador -->
<a href="/ruta-antigua" (click)="navegar($event); $event.preventDefault()">Link manejado</a>
```

**Detalle interno**: Angular registra los event listeners usando `Renderer2.listen()` internamente, lo que garantiza compatibilidad con SSR. Cuando Zone.js detecta un evento del DOM, dispara un ciclo de detección de cambios desde el componente raíz hacia abajo. En aplicaciones Zoneless, Angular registra los listeners directamente y programa la detección de cambios únicamente para el componente afectado.

---

### 4. Enlace Bidireccional (Two-Way Binding) y la API `model()`

El Two-Way Binding sincroniza el estado entre TypeScript y la plantilla en ambas direcciones. Tradicionalmente se implementaba con `[(ngModel)]` del `FormsModule`.

Angular v17.2+ introduce la API **`model()`**, que permite crear Two-Way Binding nativo sin dependencia del `FormsModule`:

```typescript
import { Component, model } from "@angular/core";

@Component({
  selector: "ui-slider",
  standalone: true,
  template: `
    <div class="slider-container">
      <input type="range" [min]="min()" [max]="max()"
             [value]="valor()"
             (input)="valor.set(+$any($event.target).value)">
      <span class="valor-display">{{ valor() }}</span>
    </div>
  `
})
export class SliderComponent {
  // model() crea un Signal bidireccional que se sincroniza con el padre automáticamente
  readonly valor = model<number>(50);
  readonly min = model<number>(0);
  readonly max = model<number>(100);
}
```

Uso en el componente padre:
```html
<!-- El valor del slider se sincroniza bidireccionalmente con 'volumen' -->
<ui-slider [(valor)]="volumen" [(min)]="volumenMin"></ui-slider>
<p>Volumen actual: {{ volumen }}</p>
```

**¿Cómo funciona internamente?** `model()` genera bajo el capó un Signal de lectura/escritura Y un output con el nombre `<propiedad>Change`. Cuando el hijo llama a `valor.set(80)`, Angular automáticamente emite el evento `valorChange` con el nuevo valor, y el padre actualiza su binding. Es exactamente la convención `[(x)]` = `[x]` + `(xChange)`, pero sin boilerplate.

---

## 3.2 Pipes: Transformación de Datos en Plantillas

Los **Pipes** son funciones de transformación declarativa que se aplican directamente en las expresiones de las plantillas usando el operador `|`. Angular incluye pipes integrados para formateo de fechas, números, monedas, y manipulación de strings y arrays.

### Pipes Puros vs. Impuros: Una Distinción de Rendimiento Crítica

| Tipo | Ejecución | Rendimiento | Uso |
|---|---|---|---|
| **Puro** (por defecto) | Solo se re-ejecuta cuando la **referencia** de la entrada cambia | Excelente: memoriza el resultado | Formateo de fechas, monedas, transformaciones determinísticas |
| **Impuro** (`pure: false`) | Se re-ejecuta en **cada** ciclo de detección de cambios | Peligroso: puede causar cuellos de botella | Filtros que dependen de estado mutable externo |

```typescript
import { Pipe, PipeTransform } from "@angular/core";

@Pipe({
  name: "resaltarBusqueda",
  standalone: true,
  pure: true // DEFAULT: solo se recalcula si 'valor' o 'termino' cambian de referencia
})
export class ResaltarBusquedaPipe implements PipeTransform {
  transform(valor: string, termino: string): string {
    if (!termino || !valor) return valor;
    
    // Escapar caracteres especiales de regex para prevenir inyección
    const terminoEscapado = termino.replace(/[.*+?^${}()|[\]\\]/g, "\\$&");
    const regex = new RegExp(`(${terminoEscapado})`, "gi");
    
    return valor.replace(regex, `<mark class="bg-yellow-200">$1</mark>`);
  }
}
```

Uso en la plantilla:
```html
<!-- Angular sanitiza el HTML por defecto, necesitamos [innerHTML] -->
<p [innerHTML]="producto().nombre | resaltarBusqueda:terminoBusqueda()"></p>
```

> **Regla de oro**: Nunca uses un pipe impuro para operaciones costosas (filtrar, ordenar, buscar). En su lugar, usa un `computed()` Signal en la clase del componente que derive la lista filtrada. Los pipes impuros se re-ejecutan en CADA ciclo de detección, incluso si los datos no han cambiado.

---

## 3.3 Directivas Clásicas frente al Control Flow Moderno

Históricamente, Angular dependía de **directivas estructurales** (precedidas por `*`) para controlar el renderizado condicional y los bucles. Las más populares eran `*ngIf`, `*ngFor` y `*ngSwitch`, todas parte del `CommonModule`.

### Problemas de las Directivas Estructurales Clásicas

1. **Dependencia de `CommonModule`**: Obligatorio importar un módulo pesado solo para condicionales básicos.
2. **Micro-sintaxis opaca**: Expresiones como `*ngFor="let item of items; trackBy: trackFn; let i = index; let odd = odd"` eran difíciles de leer y propensas a errores tipográficos.
3. **`<ng-template>` implícitos**: Cada directiva `*` creaba un `<ng-template>` invisible bajo el capó, añadiendo overhead de memoria.
4. **Type narrowing limitado**: `*ngIf="usuario as usr"` tenía inferencia de tipos pobre comparada con el TypeScript moderno.

### La Revolución del Control Flow Sintáctico (Angular v17+)

El nuevo Control Flow usa el prefijo `@` y se integra directamente en el compilador Ivy. No requiere importaciones, genera código más eficiente y ofrece type narrowing completo.

**Detalle interno del compilador**: Cuando Ivy encuentra `@if (cargando()) { ... } @else { ... }`, no crea `<ng-template>` ni directivas. En su lugar, genera instrucciones de renderizado condicional de bajo nivel (`ɵɵconditional()`) que manipulan directamente los contenedores de vista (LContainer) del árbol de vistas de Angular. Esto elimina una capa completa de abstracción, resultando en menos objetos en memoria y menos pasos de procesamiento.

---

## 3.4 El Control Flow Sintáctico al Detalle

### 1. El Bloque `@if` — Condicionales con Type Narrowing

Sustituye completamente a `*ngIf`. Ofrece soporte nativo para `@else if` y `@else`, con inferencia de tipos automática.

```html
<!-- Condicionales con ramificaciones múltiples -->
@if (cargando()) {
  <div class="flex items-center gap-2 p-4 bg-blue-50 rounded">
    <div class="animate-spin h-5 w-5 border-2 border-blue-500 rounded-full border-t-transparent"></div>
    <span>Consultando servidores de la empresa...</span>
  </div>
} @else if (error()) {
  <div class="p-4 bg-red-50 border border-red-200 rounded">
    <h3 class="font-bold text-red-800">Error de Conexión</h3>
    <p class="text-red-600">{{ error()!.message }}</p>
    <button (click)="reintentar()" class="mt-2 px-3 py-1 bg-red-600 text-white rounded">
      Reintentar
    </button>
  </div>
} @else if (datos()?.length === 0) {
  <div class="p-8 text-center text-gray-500">
    <p>No se encontraron resultados para tu búsqueda.</p>
  </div>
} @else {
  <div class="grid grid-cols-3 gap-4">
    @for (item of datos(); track item.id) {
      <app-tarjeta-producto [producto]="item"></app-tarjeta-producto>
    }
  </div>
}
```

**Type narrowing automático**: Dentro del bloque `@if`, Angular infiere correctamente que la condición es verdadera. Si escribes `@if (usuario()) { <p>{{ usuario()!.nombre }}</p> }`, el compilador sabe que `usuario()` no es `null` dentro del bloque.

---

### 2. El Bloque `@for` — Bucles con Reconciliación Eficiente

Sustituye a `*ngFor` con un rendimiento drásticamente superior gracias al **algoritmo de reconciliación basado en `track`**.

```html
<table class="w-full border-collapse">
  <thead>
    <tr>
      <th class="border p-2 text-left">#</th>
      <th class="border p-2 text-left">Producto</th>
      <th class="border p-2 text-right">Precio</th>
      <th class="border p-2 text-center">Acciones</th>
    </tr>
  </thead>
  <tbody>
    @for (producto of productosFiltrados(); track producto.id; let idx = $index) {
      <tr [class.bg-gray-50]="$even" [class.bg-white]="$odd">
        <td class="border p-2">{{ idx + 1 }}</td>
        <td class="border p-2">
          <span class="font-semibold">{{ producto.nombre }}</span>
          @if ($first) {
            <span class="ml-2 text-xs bg-green-100 text-green-800 px-1 rounded">Más vendido</span>
          }
        </td>
        <td class="border p-2 text-right">{{ producto.precio | currency:'EUR' }}</td>
        <td class="border p-2 text-center">
          <button (click)="editar(producto)" class="text-blue-600">Editar</button>
        </td>
      </tr>
    } @empty {
      <tr>
        <td colspan="4" class="p-8 text-center text-gray-400">
          No hay productos en el catálogo que coincidan con los filtros seleccionados.
        </td>
      </tr>
    }
  </tbody>
</table>
```

#### Variables Contextuales Disponibles

| Variable | Tipo | Descripción |
|---|---|---|
| `$index` | `number` | Índice basado en cero del elemento actual |
| `$count` | `number` | Número total de elementos de la colección |
| `$first` | `boolean` | `true` si es el primer elemento |
| `$last` | `boolean` | `true` si es el último elemento |
| `$even` | `boolean` | `true` si el índice es par |
| `$odd` | `boolean` | `true` si el índice es impar |

#### El Algoritmo de Reconciliación de `track` en Profundidad

Cuando la lista de datos cambia (agregar, eliminar, reordenar), Angular necesita determinar qué elementos del DOM deben ser creados, destruidos o movidos. El valor de `track` sirve como **clave de identidad única** que Angular utiliza para asociar cada dato con su representación visual en el DOM.

```
Estado Anterior:  [A:1] [B:2] [C:3] [D:4]
                   ↓     ↓     ↓     ↓
DOM:              <li>A  <li>B  <li>C  <li>D

Estado Nuevo:     [C:3] [A:1] [E:5] [D:4]   (B eliminado, E agregado, C y A reordenados)

Con track por ID:
  - C:3 → Mover <li>C a posición 0   (NO destruir/recrear)
  - A:1 → Mover <li>A a posición 1   (NO destruir/recrear)
  - E:5 → Crear nuevo <li>E          (Nuevo nodo DOM)
  - B:2 → Destruir <li>B             (Nodo eliminado)
  - D:4 → Mantener <li>D en posición (Sin cambios)

Sin track (o track por $index):
  - TODOS los <li> se destruyen y se recrean desde cero
  - 4 destrucciones + 4 creaciones = INEFICIENTE
```

> **Regla mandatoria**: `track` es **obligatorio** en `@for`. Usa siempre una propiedad de identidad única del objeto (como `id`, `uuid`, `sku`). Usar `track $index` es funcionalmente equivalente a no trackear y anula la optimización. Solo es aceptable para arrays de primitivos que nunca cambian de orden.

---

### 3. El Bloque `@switch` — Bifurcaciones Múltiples

Sustituye a la verbosa combinación de `*ngSwitch` / `*ngSwitchCase` / `*ngSwitchDefault`:

```html
@switch (pedido().estado) {
  @case ('pendiente') {
    <div class="flex items-center gap-2 text-amber-600">
      <span class="animate-pulse">●</span>
      <span>Pedido pendiente de procesamiento</span>
    </div>
  }
  @case ('enviado') {
    <div class="flex items-center gap-2 text-blue-600">
      <span>📦</span>
      <span>En tránsito — Tracking: {{ pedido().codigoSeguimiento }}</span>
    </div>
  }
  @case ('entregado') {
    <div class="flex items-center gap-2 text-green-600">
      <span>✅</span>
      <span>Entregado el {{ pedido().fechaEntrega | date:'dd/MM/yyyy' }}</span>
    </div>
  }
  @default {
    <div class="text-gray-500">Estado desconocido: {{ pedido().estado }}</div>
  }
}
```

---

## 3.5 Template Reference Variables y Signal Queries

Las **Template Reference Variables** (`#ref`) permiten obtener una referencia directa a un elemento nativo del DOM o a una instancia de componente hijo desde la plantilla:

```html
<input #buscadorInput type="text" placeholder="Escribe aquí...">
<button (click)="buscar(buscadorInput.value)">Buscar</button>

<!-- Referencia a un componente hijo -->
<app-reproductor-video #player></app-reproductor-video>
<button (click)="player.pausar()">Pausar Vídeo</button>
```

### Signal Queries: `viewChild()` y `viewChildren()` (Angular v17.2+)

Las nuevas Signal Queries reemplazan los decoradores clásicos `@ViewChild` y `@ViewChildren` con APIs basadas en Signals que se integran perfectamente con `computed()` y `effect()`:

```typescript
import { Component, viewChild, viewChildren, ElementRef, AfterViewInit, computed } from "@angular/core";
import { GraficoComponent } from "./grafico.component";

@Component({
  selector: "app-dashboard-analytics",
  standalone: true,
  imports: [GraficoComponent],
  template: `
    <div class="dashboard">
      <input #buscador type="text" placeholder="Filtrar...">
      
      <app-grafico titulo="Ventas Q1"></app-grafico>
      <app-grafico titulo="Ventas Q2"></app-grafico>
      <app-grafico titulo="Ventas Q3"></app-grafico>
    </div>
  `
})
export class DashboardAnalyticsComponent {
  // Signal Query: obtener una referencia a un elemento nativo del DOM
  private readonly buscadorRef = viewChild<ElementRef<HTMLInputElement>>("buscador");
  
  // Signal Query: obtener una referencia a un componente hijo
  private readonly primerGrafico = viewChild(GraficoComponent);
  
  // Signal Query: obtener TODAS las instancias de un componente hijo
  private readonly todosLosGraficos = viewChildren(GraficoComponent);
  
  // Computed derivado de la query
  readonly cantidadGraficos = computed(() => this.todosLosGraficos().length);

  enfocarBuscador(): void {
    // El Signal ya contiene la referencia actualizada (o undefined si no existe)
    this.buscadorRef()?.nativeElement.focus();
  }

  actualizarTodosLosGraficos(): void {
    this.todosLosGraficos().forEach(grafico => grafico.refrescar());
  }
}
```

---

## 3.6 Proyección de Contenido (`ng-content`)

La **proyección de contenido** permite que un componente padre inserte HTML arbitrario dentro de la plantilla de un componente hijo. Es el equivalente Angular de los `children` de React o los `slots` de Vue.

### Proyección Simple

```typescript
@Component({
  selector: "ui-card",
  standalone: true,
  template: `
    <div class="border rounded-lg shadow-md overflow-hidden bg-white">
      <div class="p-4 bg-gray-50 border-b font-bold text-lg">
        <ng-content select="[card-header]"></ng-content>
      </div>
      <div class="p-6">
        <ng-content></ng-content>
      </div>
      <div class="p-3 bg-gray-50 border-t text-right">
        <ng-content select="[card-footer]"></ng-content>
      </div>
    </div>
  `
})
export class CardComponent {}
```

Uso por el componente padre:
```html
<ui-card>
  <h2 card-header>Detalles del Producto</h2>
  
  <!-- Este contenido se proyecta en el <ng-content> sin selector -->
  <p>Descripción completa del producto con todas sus características técnicas.</p>
  <img [src]="producto().imagen" [alt]="producto().nombre">
  
  <div card-footer>
    <button (click)="comprar()" class="px-4 py-2 bg-blue-600 text-white rounded">Comprar</button>
  </div>
</ui-card>
```

### Proyección Condicional con `@ContentChild` y Signal Queries

A veces necesitamos saber si el padre proyectó contenido en un slot específico para renderizar una sección condicionalmente:

```typescript
import { Component, contentChild, TemplateRef } from "@angular/core";

@Component({
  selector: "ui-expandable-panel",
  standalone: true,
  template: `
    <div class="border rounded">
      <div class="p-4 cursor-pointer" (click)="alternarExpandido()">
        <ng-content select="[panel-titulo]"></ng-content>
      </div>
      
      @if (expandido) {
        <div class="p-4 border-t">
          <ng-content></ng-content>
        </div>
        
        <!-- Renderizar el footer SOLO si el padre proyectó uno -->
        @if (footerTemplate()) {
          <div class="p-3 bg-gray-50 border-t">
            <ng-container [ngTemplateOutlet]="footerTemplate()!"></ng-container>
          </div>
        }
      }
    </div>
  `
})
export class ExpandablePanelComponent {
  expandido = false;
  
  // Signal Query para detectar si el padre proyectó un template de footer
  readonly footerTemplate = contentChild<TemplateRef<unknown>>("panelFooter");
  
  alternarExpandido(): void {
    this.expandido = !this.expandido;
  }
}
```

---

## 3.7 Directivas Personalizadas Avanzadas

### Directiva de Atributo con `Renderer2` y Signals

```typescript
import { Directive, ElementRef, Renderer2, input, effect, OnInit } from "@angular/core";

@Directive({
  selector: "[uiResaltado]",
  standalone: true
})
export class ResaltadoDirective implements OnInit {
  readonly colorResaltado = input<string>("rgba(59, 130, 246, 0.15)");
  readonly colorOriginal = input<string>("transparent");

  constructor(
    private readonly el: ElementRef,
    private readonly renderer: Renderer2
  ) {
    // Usar effect() para reaccionar a cambios en los inputs de forma reactiva
    effect(() => {
      // Este código se re-ejecuta automáticamente si colorOriginal() cambia
      this.renderer.setStyle(this.el.nativeElement, "background-color", this.colorOriginal());
    });
  }

  ngOnInit(): void {
    this.renderer.setStyle(this.el.nativeElement, "transition", "background-color 0.3s ease");
    this.renderer.listen(this.el.nativeElement, "mouseenter", () => {
      this.renderer.setStyle(this.el.nativeElement, "background-color", this.colorResaltado());
    });
    this.renderer.listen(this.el.nativeElement, "mouseleave", () => {
      this.renderer.setStyle(this.el.nativeElement, "background-color", this.colorOriginal());
    });
  }
}
```

### La API `hostDirectives`: Composición de Directivas (Angular v15+)

`hostDirectives` permite componer múltiples directivas sobre un componente sin que el consumidor las conozca ni las aplique manualmente. Es el patrón de **composición sobre herencia** aplicado a las directivas:

```typescript
import { Directive, HostListener } from "@angular/core";

// Directiva de accesibilidad: añade soporte de teclado
@Directive({ selector: "[a11yKeyboard]", standalone: true })
export class KeyboardA11yDirective {
  @HostListener("keydown.enter", ["$event"])
  @HostListener("keydown.space", ["$event"])
  onActivar(event: KeyboardEvent): void {
    event.preventDefault();
    (event.target as HTMLElement).click();
  }
}

// Directiva de analytics: registra interacciones
@Directive({ selector: "[analytics]", standalone: true })
export class AnalyticsDirective {
  @HostListener("click")
  onClick(): void {
    console.log("[Analytics] Interacción registrada:", new Date().toISOString());
  }
}

// Componente que COMPONE ambas directivas automáticamente
@Component({
  selector: "ui-action-button",
  standalone: true,
  hostDirectives: [
    KeyboardA11yDirective,  // Se aplica automáticamente sin que el padre la declare
    AnalyticsDirective       // Se aplica automáticamente sin que el padre la declare
  ],
  template: `<button [attr.role]="'button'"><ng-content></ng-content></button>`
})
export class ActionButtonComponent {}
```

Ahora, cada vez que alguien use `<ui-action-button>`, automáticamente tendrá soporte de teclado y tracking de analytics **sin importar ni declarar nada adicional**.

---

## 3.8 Sanitización y Seguridad contra XSS

Angular sanitiza automáticamente todo el contenido interpolado para prevenir ataques **Cross-Site Scripting (XSS)**. Cuando insertas HTML dinámico via `[innerHTML]`, Angular elimina cualquier etiqueta o atributo potencialmente peligroso (`<script>`, `onclick`, `onerror`, etc.).

Sin embargo, hay escenarios legítimos donde necesitas insertar HTML, CSS o URLs que Angular marca como inseguros. Para estos casos, el servicio **`DomSanitizer`** permite marcar contenido como confiable de forma explícita:

```typescript
import { Component, inject, signal } from "@angular/core";
import { DomSanitizer, SafeHtml } from "@angular/platform-browser";

@Component({
  selector: "app-editor-preview",
  standalone: true,
  template: `
    <div class="editor-preview" [innerHTML]="htmlSeguro()"></div>
  `
})
export class EditorPreviewComponent {
  private readonly sanitizer = inject(DomSanitizer);
  
  // HTML crudo proveniente del editor de texto enriquecido del usuario
  private readonly htmlCrudo = signal<string>('<h1>Título</h1><p>Contenido <em>con formato</em></p>');
  
  // Marcar como confiable SOLO si el HTML proviene de una fuente controlada
  readonly htmlSeguro = signal<SafeHtml>(
    this.sanitizer.bypassSecurityTrustHtml(this.htmlCrudo())
  );
}
```

> **ADVERTENCIA DE SEGURIDAD CRÍTICA**: `bypassSecurityTrustHtml()` desactiva TODA la sanitización de Angular para ese contenido. Si el HTML proviene de input del usuario sin sanitización previa en el servidor, estás abriendo una vulnerabilidad XSS directa. Solo usa esta API con contenido generado internamente o sanitizado por tu backend.

---

## Resumen del Capítulo

* La plantilla Angular es un **DSL compilado** por Ivy a instrucciones de renderizado de bajo nivel que manipulan directamente el DOM, sin Virtual DOM intermedio.
* La distinción entre **atributo HTML** (estático, del markup) y **propiedad DOM** (dinámica, del motor del navegador) es fundamental para entender cuándo usar `[propiedad]` vs `[attr.atributo]`.
* El **Control Flow Sintáctico** (`@if`, `@for`, `@switch`) elimina la necesidad de `CommonModule`, genera código hasta un **90% más eficiente** y ofrece type narrowing automático.
* El algoritmo de reconciliación de **`track`** en `@for` utiliza claves de identidad para reutilizar nodos DOM existentes, eliminando repintadas destructivas.
* Los **Pipes puros** se memorizan automáticamente y solo se re-ejecutan ante cambios de referencia. Los **pipes impuros** se ejecutan en cada ciclo de detección y deben evitarse para operaciones costosas.
* La API **`model()`** proporciona Two-Way Binding nativo basado en Signals sin dependencia del `FormsModule`.
* Las **Signal Queries** (`viewChild()`, `viewChildren()`, `contentChild()`) reemplazan los decoradores clásicos con APIs reactivas.
* **`hostDirectives`** implementa composición de directivas, permitiendo aplicar comportamientos transversales (accesibilidad, analytics) sin que el consumidor los conozca.
* **`DomSanitizer`** es la última línea de defensa contra XSS, pero `bypassSecurityTrust*()` debe usarse con extrema cautela.

En el próximo capítulo, nos adentraremos en el concepto más revolucionario del Angular Renaissance: **Angular Signals**, el nuevo motor reactivo que redefine cómo fluye la información y que abre la puerta a aplicaciones completamente libres de Zone.js.

---

← [Capítulo anterior](02-arquitectura-y-componentes.md) | [Inicio](README.md) | [Capítulo siguiente →](04-signals.md)
