# Capítulo 17: Angular Material, CDK y Animaciones

> "Angular Material no es una librería de componentes bonitos. Es la implementación oficial de las guías de Material Design 3, con accesibilidad WCAG incluida de serie, temas personalizables y un CDK headless para cuando quieras construir tus propios componentes desde cero."

Cuando construyes aplicaciones empresariales, reinventar la rueda para cada componente de UI (tablas, diálogos, menús, autocompletado, drag & drop) es una inversión de tiempo que rara vez se justifica. **Angular Material** proporciona un catálogo de más de 30 componentes probados en producción por millones de aplicaciones, con accesibilidad integrada, soporte de teclado, temas personalizables y compatibilidad total con Signals y Standalone.

Debajo de Angular Material vive el **Component Dev Kit (CDK)**, una capa de utilidades "headless" (sin estilos visuales) que resuelven problemas complejos de interacción: overlays flotantes, scrolling virtual, drag & drop, gestión de foco y detección de plataforma. El CDK te permite construir tus propios componentes de UI con la misma robustez de Angular Material pero con tu diseño personalizado.

Además, Angular incluye un potente **sistema de animaciones** basado en Web Animations API que permite declarar transiciones complejas de forma declarativa dentro de los metadatos del componente.

---

## 17.1 Angular Material: Instalación y Sistema de Temas

### Instalación

```bash
ng add @angular/material
```

El schematic te preguntará:
- **Tema**: Selecciona `Custom` para máxima flexibilidad.
- **Typography**: Sí, para tipografía de Material Design 3.
- **Animations**: Sí, para habilitar el módulo de animaciones.

### Sistema de Temas con SCSS (Material Design 3)

Angular Material 3 utiliza un sistema de tokens de diseño basado en SCSS. Crea tu tema personalizado:

```scss
// src/styles.scss
@use "@angular/material" as mat;

// Definir tema personalizado con colores de la marca
$mi-tema: mat.define-theme((
  color: (
    theme-type: light,
    primary: mat.$azure-palette,
    tertiary: mat.$violet-palette,
  ),
  typography: (
    brand-family: "Inter, sans-serif",
    plain-family: "Roboto, sans-serif",
  ),
  density: (
    scale: 0,
  )
));

// Tema oscuro
$mi-tema-oscuro: mat.define-theme((
  color: (
    theme-type: dark,
    primary: mat.$cyan-palette,
    tertiary: mat.$magenta-palette,
  )
));

// Aplicar el tema
html {
  @include mat.all-component-themes($mi-tema);
  
  // Tema oscuro condicional
  &[data-theme="dark"] {
    @include mat.all-component-colors($mi-tema-oscuro);
  }
}
```

---

## 17.2 Componentes Esenciales de Angular Material

### Tablas con `MatTable`

```typescript
import { Component, signal } from "@angular/core";
import { MatTableModule } from "@angular/material/table";
import { MatSortModule, Sort } from "@angular/material/sort";
import { MatPaginatorModule, PageEvent } from "@angular/material/paginator";
import { MatInputModule } from "@angular/material/input";

interface Empleado {
  id: number;
  nombre: string;
  departamento: string;
  salario: number;
}

@Component({
  selector: "app-tabla-empleados",
  standalone: true,
  imports: [MatTableModule, MatSortModule, MatPaginatorModule, MatInputModule],
  template: `
    <mat-form-field appearance="outline" class="w-full mb-4">
      <mat-label>Buscar empleado</mat-label>
      <input matInput (input)="filtrar($event)" placeholder="Nombre o departamento">
    </mat-form-field>

    <table mat-table [dataSource]="datosFiltrados()" matSort (matSortChange)="ordenar($event)">
      
      <ng-container matColumnDef="nombre">
        <th mat-header-cell *matHeaderCellDef mat-sort-header>Nombre</th>
        <td mat-cell *matCellDef="let emp">{{ emp.nombre }}</td>
      </ng-container>
      
      <ng-container matColumnDef="departamento">
        <th mat-header-cell *matHeaderCellDef mat-sort-header>Departamento</th>
        <td mat-cell *matCellDef="let emp">{{ emp.departamento }}</td>
      </ng-container>
      
      <ng-container matColumnDef="salario">
        <th mat-header-cell *matHeaderCellDef mat-sort-header>Salario</th>
        <td mat-cell *matCellDef="let emp">{{ emp.salario | currency:'EUR' }}</td>
      </ng-container>

      <tr mat-header-row *matHeaderRowDef="['nombre', 'departamento', 'salario']"></tr>
      <tr mat-row *matRowDef="let row; columns: ['nombre', 'departamento', 'salario']"></tr>
    </table>

    <mat-paginator [length]="totalEmpleados()" [pageSize]="10" [pageSizeOptions]="[5, 10, 25]"
                   (page)="cambiarPagina($event)">
    </mat-paginator>
  `
})
export class TablaEmpleadosComponent {
  private readonly empleados = signal<Empleado[]>([
    { id: 1, nombre: "Ana García", departamento: "Ingeniería", salario: 65000 },
    { id: 2, nombre: "Carlos Ruiz", departamento: "Marketing", salario: 48000 },
    { id: 3, nombre: "Laura Martínez", departamento: "Ingeniería", salario: 72000 },
    // ... más datos
  ]);
  
  readonly datosFiltrados = signal<Empleado[]>(this.empleados());
  readonly totalEmpleados = signal(this.empleados().length);

  filtrar(event: Event): void {
    const termino = (event.target as HTMLInputElement).value.toLowerCase();
    this.datosFiltrados.set(
      this.empleados().filter(e =>
        e.nombre.toLowerCase().includes(termino) ||
        e.departamento.toLowerCase().includes(termino)
      )
    );
  }

  ordenar(sort: Sort): void {
    const datos = [...this.datosFiltrados()];
    datos.sort((a, b) => {
      const asc = sort.direction === "asc" ? 1 : -1;
      return (a[sort.active as keyof Empleado] > b[sort.active as keyof Empleado] ? 1 : -1) * asc;
    });
    this.datosFiltrados.set(datos);
  }

  cambiarPagina(event: PageEvent): void {
    console.log("Página:", event.pageIndex, "Tamaño:", event.pageSize);
  }
}
```

### Diálogos con `MatDialog`

```typescript
import { Component, inject } from "@angular/core";
import { MatDialog, MatDialogModule } from "@angular/material/dialog";
import { MatButtonModule } from "@angular/material/button";

@Component({
  selector: "app-confirmar-dialog",
  standalone: true,
  imports: [MatDialogModule, MatButtonModule],
  template: `
    <h2 mat-dialog-title>Confirmar Acción</h2>
    <mat-dialog-content>
      <p>¿Estás seguro de que deseas eliminar este elemento? Esta acción no se puede deshacer.</p>
    </mat-dialog-content>
    <mat-dialog-actions align="end">
      <button mat-button mat-dialog-close>Cancelar</button>
      <button mat-flat-button color="warn" [mat-dialog-close]="true">Eliminar</button>
    </mat-dialog-actions>
  `
})
export class ConfirmarDialogComponent {}

// Uso desde el componente que abre el diálogo:
@Component({
  selector: "app-gestion",
  standalone: true,
  imports: [MatButtonModule],
  template: `<button mat-raised-button color="warn" (click)="confirmarEliminacion()">Eliminar</button>`
})
export class GestionComponent {
  private readonly dialog = inject(MatDialog);

  confirmarEliminacion(): void {
    const dialogRef = this.dialog.open(ConfirmarDialogComponent, {
      width: "400px",
      disableClose: true // No se cierra al hacer clic fuera
    });

    dialogRef.afterClosed().subscribe(confirmado => {
      if (confirmado) {
        console.log("Elemento eliminado");
      }
    });
  }
}
```

### Snackbar para Notificaciones

```typescript
import { inject } from "@angular/core";
import { MatSnackBar } from "@angular/material/snack-bar";

// Dentro de un componente o servicio:
private readonly snackBar = inject(MatSnackBar);

mostrarExito(): void {
  this.snackBar.open("Producto agregado al carrito", "Deshacer", {
    duration: 5000,
    horizontalPosition: "end",
    verticalPosition: "top",
    panelClass: ["snackbar-exito"]
  });
}
```

---

## 17.3 Angular CDK: Herramientas Headless

El CDK proporciona las primitivas de comportamiento sin opinión visual.

### Virtual Scrolling (Listas de Miles de Elementos)

```typescript
import { Component, signal } from "@angular/core";
import { ScrollingModule } from "@angular/cdk/scrolling";

@Component({
  selector: "app-lista-masiva",
  standalone: true,
  imports: [ScrollingModule],
  template: `
    <!-- Solo renderiza ~20 elementos visibles, aunque haya 100.000 -->
    <cdk-virtual-scroll-viewport itemSize="48" class="h-96 border rounded">
      <div *cdkVirtualFor="let item of items()" class="h-12 flex items-center px-4 border-b">
        <span class="font-mono text-sm">#{{ item.id }}</span>
        <span class="ml-4">{{ item.nombre }}</span>
      </div>
    </cdk-virtual-scroll-viewport>
  `
})
export class ListaMasivaComponent {
  readonly items = signal(
    Array.from({ length: 100_000 }, (_, i) => ({
      id: i + 1,
      nombre: `Elemento ${i + 1}`
    }))
  );
}
```

### Drag & Drop

```typescript
import { Component, signal } from "@angular/core";
import { CdkDragDrop, DragDropModule, moveItemInArray, transferArrayItem } from "@angular/cdk/drag-drop";

@Component({
  selector: "app-kanban-board",
  standalone: true,
  imports: [DragDropModule],
  template: `
    <div class="flex gap-4">
      <!-- Columna: Por hacer -->
      <div class="flex-1 bg-gray-50 p-4 rounded-lg">
        <h3 class="font-bold mb-3">📋 Por hacer</h3>
        <div cdkDropList [cdkDropListData]="porHacer()" (cdkDropListDropped)="soltar($event)"
             [cdkDropListConnectedTo]="['en-progreso']" class="min-h-32 space-y-2">
          @for (tarea of porHacer(); track tarea) {
            <div cdkDrag class="bg-white p-3 rounded shadow cursor-move border-l-4 border-blue-500">
              {{ tarea }}
            </div>
          }
        </div>
      </div>

      <!-- Columna: En progreso -->
      <div class="flex-1 bg-gray-50 p-4 rounded-lg">
        <h3 class="font-bold mb-3">🔄 En progreso</h3>
        <div cdkDropList id="en-progreso" [cdkDropListData]="enProgreso()"
             (cdkDropListDropped)="soltar($event)"
             [cdkDropListConnectedTo]="['completado']" class="min-h-32 space-y-2">
          @for (tarea of enProgreso(); track tarea) {
            <div cdkDrag class="bg-white p-3 rounded shadow cursor-move border-l-4 border-amber-500">
              {{ tarea }}
            </div>
          }
        </div>
      </div>
    </div>
  `
})
export class KanbanBoardComponent {
  readonly porHacer = signal(["Diseñar API", "Escribir tests", "Code review"]);
  readonly enProgreso = signal(["Implementar login"]);

  soltar(event: CdkDragDrop<string[]>): void {
    if (event.previousContainer === event.container) {
      const arr = [...event.container.data];
      moveItemInArray(arr, event.previousIndex, event.currentIndex);
      // Actualizar el signal correspondiente
    } else {
      const prev = [...event.previousContainer.data];
      const curr = [...event.container.data];
      transferArrayItem(prev, curr, event.previousIndex, event.currentIndex);
    }
  }
}
```

### Overlay (Tooltips, Dropdowns, Popovers Personalizados)

```typescript
import { Component, inject, ViewContainerRef } from "@angular/core";
import { Overlay, OverlayModule, OverlayRef } from "@angular/cdk/overlay";
import { ComponentPortal } from "@angular/cdk/portal";

@Component({
  selector: "app-tooltip-personalizado",
  standalone: true,
  imports: [OverlayModule],
  template: `
    <button (mouseenter)="mostrar($event)" (mouseleave)="ocultar()" class="px-4 py-2 bg-blue-600 text-white rounded">
      Pasa el ratón
    </button>
  `
})
export class TooltipPersonalizadoComponent {
  private readonly overlay = inject(Overlay);
  private overlayRef: OverlayRef | null = null;

  mostrar(event: MouseEvent): void {
    const positionStrategy = this.overlay.position()
      .flexibleConnectedTo(event.target as HTMLElement)
      .withPositions([{
        originX: "center", originY: "bottom",
        overlayX: "center", overlayY: "top",
        offsetY: 8
      }]);

    this.overlayRef = this.overlay.create({
      positionStrategy,
      hasBackdrop: false,
      scrollStrategy: this.overlay.scrollStrategies.close()
    });

    // Inyectar contenido dinámico
    const portal = new ComponentPortal(TooltipContentComponent);
    this.overlayRef.attach(portal);
  }

  ocultar(): void {
    this.overlayRef?.detach();
  }
}
```

---

## 17.4 Sistema de Animaciones de Angular

Angular integra un sistema de animaciones declarativo basado en Web Animations API. Las animaciones se definen como metadatos del componente y pueden reaccionar a cambios de estado, entradas/salidas del DOM y transiciones de ruta.

### Instalación

```bash
# Ya incluido si seleccionaste "Animations" durante ng add @angular/material
# Si no, provéelo manualmente:
```

```typescript
// app.config.ts
import { provideAnimationsAsync } from "@angular/platform-browser/animations/async";

export const appConfig: ApplicationConfig = {
  providers: [
    provideAnimationsAsync() // Carga las animaciones de forma perezosa
  ]
};
```

### Animaciones de Entrada y Salida

```typescript
import { Component, signal } from "@angular/core";
import { trigger, transition, style, animate, query, stagger } from "@angular/animations";

@Component({
  selector: "app-lista-animada",
  standalone: true,
  animations: [
    // Animación de aparición/desaparición individual
    trigger("fadeSlide", [
      transition(":enter", [
        style({ opacity: 0, transform: "translateY(-20px)" }),
        animate("300ms ease-out", style({ opacity: 1, transform: "translateY(0)" }))
      ]),
      transition(":leave", [
        animate("200ms ease-in", style({ opacity: 0, transform: "translateX(100px)" }))
      ])
    ]),
    
    // Animación escalonada para listas (stagger)
    trigger("listaAnimada", [
      transition("* => *", [
        query(":enter", [
          style({ opacity: 0, transform: "translateY(15px)" }),
          stagger("50ms", [
            animate("300ms ease-out", style({ opacity: 1, transform: "translateY(0)" }))
          ])
        ], { optional: true })
      ])
    ])
  ],
  template: `
    <button (click)="agregarItem()" class="mb-4 px-4 py-2 bg-green-600 text-white rounded">
      Agregar elemento
    </button>

    <div [@listaAnimada]="items().length">
      @for (item of items(); track item.id) {
        <div @fadeSlide class="p-3 mb-2 bg-white rounded shadow border-l-4 border-green-500 flex justify-between">
          <span>{{ item.texto }}</span>
          <button (click)="eliminar(item.id)" class="text-red-500 text-sm">✕</button>
        </div>
      }
    </div>
  `
})
export class ListaAnimadaComponent {
  private counter = 0;
  readonly items = signal<{ id: number; texto: string }[]>([]);

  agregarItem(): void {
    this.counter++;
    this.items.update(list => [...list, { id: this.counter, texto: `Elemento ${this.counter}` }]);
  }

  eliminar(id: number): void {
    this.items.update(list => list.filter(i => i.id !== id));
  }
}
```

### Animaciones de Transición entre Estados

```typescript
trigger("estadoPedido", [
  state("pendiente", style({ backgroundColor: "#fef3c7", transform: "scale(1)" })),
  state("procesando", style({ backgroundColor: "#dbeafe", transform: "scale(1.02)" })),
  state("completado", style({ backgroundColor: "#d1fae5", transform: "scale(1)" })),
  
  transition("pendiente => procesando", animate("400ms ease-in-out")),
  transition("procesando => completado", animate("600ms cubic-bezier(0.25, 1, 0.5, 1)")),
  transition("* => pendiente", animate("300ms ease-out"))
])
```

### Animaciones de Ruta (Route Animations)

```typescript
// app.component.ts
import { trigger, transition, style, animate, query, group } from "@angular/animations";

const routeAnimation = trigger("routeAnimations", [
  transition("* <=> *", [
    query(":enter, :leave", [
      style({ position: "absolute", width: "100%", opacity: 0 })
    ], { optional: true }),
    group([
      query(":leave", [
        animate("300ms ease-out", style({ opacity: 0, transform: "translateX(-30px)" }))
      ], { optional: true }),
      query(":enter", [
        style({ transform: "translateX(30px)" }),
        animate("300ms 100ms ease-out", style({ opacity: 1, transform: "translateX(0)" }))
      ], { optional: true })
    ])
  ])
]);

@Component({
  animations: [routeAnimation],
  template: `
    <div [@routeAnimations]="prepareRoute(outlet)">
      <router-outlet #outlet="outlet"></router-outlet>
    </div>
  `
})
export class AppComponent {
  prepareRoute(outlet: any): string {
    return outlet?.activatedRouteData?.["animation"] ?? "";
  }
}
```

### Rendimiento de Animaciones

| Propiedad | Rendimiento | Razón |
|---|---|---|
| `transform`, `opacity` | ✅ Excelente | Solo compositor GPU, no dispara layout/paint |
| `background-color`, `color` | 🟡 Bueno | Dispara paint pero no layout |
| `width`, `height`, `top`, `left` | ❌ Malo | Dispara layout + paint + composite |
| `margin`, `padding`, `border` | ❌ Malo | Dispara layout completo |

> **Regla de oro**: Anima SOLO `transform` y `opacity`. Usa `will-change: transform` en CSS para pistas al navegador.

---

## Resumen del Capítulo

* **Angular Material** proporciona 30+ componentes empresariales con accesibilidad WCAG, temas Material Design 3 personalizables y compatibilidad total con Standalone.
* El **sistema de temas** de Material 3 usa tokens SCSS con paletas (`$azure-palette`, `$violet-palette`) y soporte nativo de modo claro/oscuro.
* **MatTable** con `MatSort` y `MatPaginator` resuelve tablas de datos empresariales con ordenamiento, filtrado y paginación.
* **MatDialog** gestiona diálogos modales con datos inyectados, cierre controlado y resultado tipado.
* El **CDK** proporciona primitivas headless: **Virtual Scrolling** para listas de 100K+ elementos, **Drag & Drop** para Kanban boards, y **Overlay** para tooltips/popovers personalizados.
* Las **animaciones de Angular** se declaran en los metadatos del componente con `trigger`, `state`, `transition` y `animate`.
* Los triggers `:enter` y `:leave` animan elementos que entran/salen del DOM (compatible con `@if` y `@for`).
* **`stagger()`** crea animaciones escalonadas elegantes en listas.
* Para rendimiento óptimo, anima exclusivamente `transform` y `opacity`, que se ejecutan en el compositor GPU sin disparar reflows.

---

← [Capítulo anterior](16-microfrontends.md) | [Inicio](README.md) | [Capítulo siguiente →](18-i18n-y-a11y.md)
