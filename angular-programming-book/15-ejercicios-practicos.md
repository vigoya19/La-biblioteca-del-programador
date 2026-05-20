# Capítulo 15: Proyecto Práctico Completo e Integrador

> "La diferencia entre conocer la teoría y dominar un framework es haber construido algo real con él. Este capítulo es ese puente."

Este capítulo es fundamentalmente diferente a los anteriores. No es una referencia teórica: es un **taller práctico guiado** donde construirás, paso a paso, una aplicación E-Commerce completa de producción. Cada ejercicio incluye el **objetivo concreto**, las **instrucciones detalladas**, el **código que debes escribir**, y **tareas de verificación** para que confirmes que tu implementación funciona correctamente antes de avanzar.

Al finalizar este capítulo, habrás construido una aplicación que demuestra dominio práctico de: Signals, NgRx Signals Store, Control Flow moderno, Formularios Reactivos tipados, `@defer`, SSR y testing.

---

## Definición del Proyecto: TechStore E-Commerce

**TechStore** es una aplicación de comercio electrónico de tecnología que permite a los usuarios navegar por un catálogo de productos, filtrar por categorías, añadir productos al carrito, aplicar descuentos y completar un proceso de compra con validaciones avanzadas.

### Funcionalidades Objetivo
- Catálogo de productos con filtros reactivos basados en Signals
- Carrito de compras persistente gestionado por NgRx Signals Store
- Formulario de checkout con validaciones síncronas y asíncronas
- Carga diferida con `@defer` para optimizar el bundle inicial
- Suite de tests unitarios y E2E
- Preparación para SSR y SEO

---

## Ejercicio 1: Andamiaje del Proyecto y Diseño Modular de Rutas

### Objetivo
Crear el proyecto Angular standalone, configurar la estructura de carpetas Feature-driven y definir las rutas principales con carga perezosa.

### Paso 1.1: Crear el proyecto

Abre tu terminal y ejecuta:

```bash
ng new techstore --style=scss --ssr=false --routing
cd techstore
```

### Paso 1.2: Crear la estructura de carpetas

Genera los componentes y servicios que necesitarás:

```bash
# Componentes de layout
ng g c core/layout/header --standalone --change-detection=OnPush
ng g c core/layout/footer --standalone --change-detection=OnPush

# Feature: Catálogo
ng g c features/catalogo/catalogo-page --standalone --change-detection=OnPush
ng g c features/catalogo/components/tarjeta-producto --standalone --change-detection=OnPush
ng g c features/catalogo/components/filtro-categorias --standalone --change-detection=OnPush

# Feature: Carrito
ng g c features/carrito/carrito-page --standalone --change-detection=OnPush
ng g c features/carrito/components/item-carrito --standalone --change-detection=OnPush

# Feature: Checkout
ng g c features/checkout/checkout-page --standalone --change-detection=OnPush

# Servicios
ng g s core/services/producto
ng g s core/services/carrito-state

# Modelos
mkdir -p src/app/core/models
```

### Paso 1.3: Definir los modelos de datos

Crea el archivo `src/app/core/models/producto.interface.ts`:

```typescript
export interface Producto {
  id: string;
  nombre: string;
  descripcion: string;
  precio: number;
  categoria: "laptops" | "monitores" | "perifericos" | "almacenamiento";
  imagenUrl: string;
  stock: number;
  destacado: boolean;
}

export interface ItemCarrito {
  producto: Producto;
  cantidad: number;
}
```

### Paso 1.4: Configurar las rutas con carga perezosa

Edita `src/app/app.routes.ts`:

```typescript
import { Routes } from "@angular/router";

export const routes: Routes = [
  { path: "", redirectTo: "catalogo", pathMatch: "full" },
  {
    path: "catalogo",
    loadComponent: () =>
      import("./features/catalogo/catalogo-page/catalogo-page.component")
        .then(m => m.CatalogoPageComponent),
    title: "TechStore — Catálogo"
  },
  {
    path: "carrito",
    loadComponent: () =>
      import("./features/carrito/carrito-page/carrito-page.component")
        .then(m => m.CarritoPageComponent),
    title: "TechStore — Mi Carrito"
  },
  {
    path: "checkout",
    loadComponent: () =>
      import("./features/checkout/checkout-page/checkout-page.component")
        .then(m => m.CheckoutPageComponent),
    title: "TechStore — Checkout"
  },
  { path: "**", redirectTo: "catalogo" }
];
```

### Paso 1.5: Configurar el layout principal

Edita `src/app/app.component.ts`:

```typescript
import { Component } from "@angular/core";
import { RouterOutlet } from "@angular/router";
import { HeaderComponent } from "./core/layout/header/header.component";
import { FooterComponent } from "./core/layout/footer/footer.component";

@Component({
  selector: "app-root",
  standalone: true,
  imports: [RouterOutlet, HeaderComponent, FooterComponent],
  template: `
    <div class="min-h-screen flex flex-col bg-gray-50">
      <app-header />
      <main class="flex-1 container mx-auto px-4 py-8">
        <router-outlet />
      </main>
      <app-footer />
    </div>
  `
})
export class AppComponent {}
```

### ✅ Verificación del Ejercicio 1

Ejecuta `ng serve` y verifica que:
- [ ] La aplicación arranca sin errores en `http://localhost:4200`
- [ ] La URL redirige a `/catalogo`
- [ ] Si navegas a `/carrito` y `/checkout`, los componentes vacíos se renderizan
- [ ] El título del navegador cambia según la ruta

---

## Ejercicio 2: Construcción del Catálogo con Filtros Reactivos

### Objetivo
Implementar un servicio de datos con productos simulados, construir tarjetas de producto con el Control Flow moderno, e implementar filtros reactivos basados en Signals y `computed()`.

### Paso 2.1: Implementar el servicio de productos

Edita `src/app/core/services/producto.service.ts`:

```typescript
import { Injectable, signal, computed } from "@angular/core";
import { Producto } from "../models/producto.interface";

@Injectable({ providedIn: "root" })
export class ProductoService {
  // Datos simulados (en producción vendrían de una API)
  private readonly _productos = signal<Producto[]>([
    {
      id: "LAP-001", nombre: "Laptop UltraBook Pro 15", descripcion: "Procesador i9, 32GB RAM, 1TB SSD NVMe",
      precio: 1299.99, categoria: "laptops", imagenUrl: "assets/laptop-pro.webp", stock: 15, destacado: true
    },
    {
      id: "LAP-002", nombre: "Laptop Developer Edition", descripcion: "AMD Ryzen 9, 64GB RAM, pantalla 4K",
      precio: 1899.99, categoria: "laptops", imagenUrl: "assets/laptop-dev.webp", stock: 8, destacado: true
    },
    {
      id: "MON-001", nombre: "Monitor Curvo 34\" UltraWide", descripcion: "3440x1440, 144Hz, HDR400, USB-C",
      precio: 549.99, categoria: "monitores", imagenUrl: "assets/monitor-curved.webp", stock: 22, destacado: false
    },
    {
      id: "MON-002", nombre: "Monitor 4K 27\" IPS", descripcion: "3840x2160, 60Hz, 99% sRGB, pivotable",
      precio: 399.99, categoria: "monitores", imagenUrl: "assets/monitor-4k.webp", stock: 30, destacado: false
    },
    {
      id: "PER-001", nombre: "Teclado Mecánico Silent RGB", descripcion: "Switches Cherry MX Silent Red, aluminio",
      precio: 149.99, categoria: "perifericos", imagenUrl: "assets/keyboard.webp", stock: 50, destacado: true
    },
    {
      id: "PER-002", nombre: "Ratón Ergonómico Wireless", descripcion: "Sensor 25K DPI, 70h batería, carga rápida",
      precio: 79.99, categoria: "perifericos", imagenUrl: "assets/mouse.webp", stock: 45, destacado: false
    },
    {
      id: "ALM-001", nombre: "SSD NVMe 2TB Gen5", descripcion: "Lectura 12.400 MB/s, escritura 11.800 MB/s",
      precio: 219.99, categoria: "almacenamiento", imagenUrl: "assets/ssd.webp", stock: 60, destacado: false
    },
    {
      id: "ALM-002", nombre: "Disco Duro Externo 4TB", descripcion: "USB 3.2 Gen 2, resistente a golpes, cifrado",
      precio: 129.99, categoria: "almacenamiento", imagenUrl: "assets/hdd.webp", stock: 35, destacado: false
    }
  ]);

  // Estado de filtros
  readonly categoriaSeleccionada = signal<string>("todas");
  readonly terminoBusqueda = signal<string>("");
  readonly ordenamiento = signal<"precio-asc" | "precio-desc" | "nombre">("nombre");

  // Productos filtrados y ordenados (computed derivado)
  readonly productosFiltrados = computed(() => {
    let resultado = this._productos();
    
    // Filtrar por categoría
    const cat = this.categoriaSeleccionada();
    if (cat !== "todas") {
      resultado = resultado.filter(p => p.categoria === cat);
    }
    
    // Filtrar por término de búsqueda
    const termino = this.terminoBusqueda().toLowerCase().trim();
    if (termino) {
      resultado = resultado.filter(p =>
        p.nombre.toLowerCase().includes(termino) ||
        p.descripcion.toLowerCase().includes(termino)
      );
    }
    
    // Ordenar
    const orden = this.ordenamiento();
    switch (orden) {
      case "precio-asc":
        return [...resultado].sort((a, b) => a.precio - b.precio);
      case "precio-desc":
        return [...resultado].sort((a, b) => b.precio - a.precio);
      case "nombre":
      default:
        return [...resultado].sort((a, b) => a.nombre.localeCompare(b.nombre));
    }
  });

  readonly totalProductos = computed(() => this.productosFiltrados().length);
  readonly productosDestacados = computed(() => this._productos().filter(p => p.destacado));
  
  readonly categorias: string[] = ["todas", "laptops", "monitores", "perifericos", "almacenamiento"];
}
```

### Paso 2.2: Construir la tarjeta de producto (Dumb Component)

Edita `src/app/features/catalogo/components/tarjeta-producto/tarjeta-producto.component.ts`:

```typescript
import { Component, input, output, ChangeDetectionStrategy } from "@angular/core";
import { CurrencyPipe } from "@angular/common";
import { Producto } from "../../../../core/models/producto.interface";

@Component({
  selector: "app-tarjeta-producto",
  standalone: true,
  imports: [CurrencyPipe],
  changeDetection: ChangeDetectionStrategy.OnPush,
  template: `
    <article class="bg-white rounded-lg shadow-md overflow-hidden hover:shadow-xl transition-shadow duration-300 flex flex-col h-full">
      <div class="aspect-video bg-gray-100 flex items-center justify-center p-4">
        <span class="text-4xl">🖥️</span>
      </div>
      
      <div class="p-4 flex flex-col flex-1">
        <div class="flex items-start justify-between gap-2 mb-2">
          <h3 class="font-bold text-gray-900 text-sm leading-tight">{{ producto().nombre }}</h3>
          @if (producto().destacado) {
            <span class="shrink-0 text-xs bg-amber-100 text-amber-800 px-2 py-0.5 rounded-full font-semibold">
              ⭐ Destacado
            </span>
          }
        </div>
        
        <p class="text-gray-500 text-xs mb-3 flex-1">{{ producto().descripcion }}</p>
        
        <div class="flex items-end justify-between mt-auto">
          <div>
            <p class="text-xl font-extrabold text-blue-600">{{ producto().precio | currency:'EUR' }}</p>
            <p class="text-xs mt-0.5" [class]="producto().stock > 10 ? 'text-green-600' : 'text-amber-600'">
              @if (producto().stock > 10) {
                En stock ({{ producto().stock }} uds.)
              } @else if (producto().stock > 0) {
                ¡Solo quedan {{ producto().stock }}!
              } @else {
                Agotado
              }
            </p>
          </div>
          
          <button
            (click)="comprar.emit(producto())"
            [disabled]="producto().stock === 0"
            class="px-4 py-2 bg-blue-600 text-white text-sm rounded-lg hover:bg-blue-700 
                   disabled:bg-gray-300 disabled:cursor-not-allowed transition-colors font-semibold">
            @if (producto().stock > 0) {
              Agregar 🛒
            } @else {
              Agotado
            }
          </button>
        </div>
      </div>
    </article>
  `
})
export class TarjetaProductoComponent {
  readonly producto = input.required<Producto>();
  readonly comprar = output<Producto>();
}
```

### Paso 2.3: Construir el componente de filtros

Edita `src/app/features/catalogo/components/filtro-categorias/filtro-categorias.component.ts`:

```typescript
import { Component, inject, ChangeDetectionStrategy } from "@angular/core";
import { ProductoService } from "../../../../core/services/producto.service";

@Component({
  selector: "app-filtro-categorias",
  standalone: true,
  changeDetection: ChangeDetectionStrategy.OnPush,
  template: `
    <div class="bg-white p-4 rounded-lg shadow-sm border space-y-4">
      <!-- Barra de búsqueda -->
      <div>
        <label for="busqueda" class="block text-sm font-semibold text-gray-700 mb-1">Buscar</label>
        <input
          id="busqueda"
          type="text"
          [value]="servicio.terminoBusqueda()"
          (input)="servicio.terminoBusqueda.set($any($event.target).value)"
          placeholder="Nombre o descripción..."
          class="w-full border rounded-lg px-3 py-2 text-sm focus:ring-2 focus:ring-blue-500 focus:border-blue-500">
      </div>
      
      <!-- Filtro de categoría -->
      <div>
        <label class="block text-sm font-semibold text-gray-700 mb-2">Categoría</label>
        <div class="flex flex-wrap gap-2">
          @for (cat of servicio.categorias; track cat) {
            <button
              (click)="servicio.categoriaSeleccionada.set(cat)"
              [class]="servicio.categoriaSeleccionada() === cat 
                ? 'bg-blue-600 text-white' 
                : 'bg-gray-100 text-gray-700 hover:bg-gray-200'"
              class="px-3 py-1.5 rounded-full text-xs font-medium transition-colors capitalize">
              {{ cat }}
            </button>
          }
        </div>
      </div>
      
      <!-- Ordenamiento -->
      <div>
        <label for="orden" class="block text-sm font-semibold text-gray-700 mb-1">Ordenar por</label>
        <select
          id="orden"
          (change)="servicio.ordenamiento.set($any($event.target).value)"
          class="w-full border rounded-lg px-3 py-2 text-sm">
          <option value="nombre">Nombre (A-Z)</option>
          <option value="precio-asc">Precio: menor a mayor</option>
          <option value="precio-desc">Precio: mayor a menor</option>
        </select>
      </div>
      
      <!-- Contador de resultados -->
      <p class="text-xs text-gray-500 text-right">
        {{ servicio.totalProductos() }} producto(s) encontrado(s)
      </p>
    </div>
  `
})
export class FiltroCategoriasComponent {
  readonly servicio = inject(ProductoService);
}
```

### Paso 2.4: Ensamblar la página del catálogo (Smart Component)

Edita `src/app/features/catalogo/catalogo-page/catalogo-page.component.ts`:

```typescript
import { Component, inject, ChangeDetectionStrategy } from "@angular/core";
import { ProductoService } from "../../../core/services/producto.service";
import { TarjetaProductoComponent } from "../components/tarjeta-producto/tarjeta-producto.component";
import { FiltroCategoriasComponent } from "../components/filtro-categorias/filtro-categorias.component";
import { Producto } from "../../../core/models/producto.interface";

@Component({
  selector: "app-catalogo-page",
  standalone: true,
  imports: [TarjetaProductoComponent, FiltroCategoriasComponent],
  changeDetection: ChangeDetectionStrategy.OnPush,
  template: `
    <div class="grid grid-cols-1 lg:grid-cols-4 gap-6">
      <!-- Sidebar de filtros -->
      <aside class="lg:col-span-1">
        <app-filtro-categorias />
      </aside>
      
      <!-- Grid de productos -->
      <section class="lg:col-span-3">
        <h1 class="text-2xl font-extrabold text-gray-900 mb-6">Catálogo de Productos</h1>
        
        <div class="grid grid-cols-1 sm:grid-cols-2 xl:grid-cols-3 gap-4">
          @for (producto of productoService.productosFiltrados(); track producto.id) {
            <app-tarjeta-producto
              [producto]="producto"
              (comprar)="agregarAlCarrito($event)" />
          } @empty {
            <div class="col-span-full text-center py-12 text-gray-400">
              <p class="text-4xl mb-4">🔍</p>
              <p class="text-lg font-semibold">No se encontraron productos</p>
              <p class="text-sm mt-1">Intenta cambiar los filtros o el término de búsqueda.</p>
            </div>
          }
        </div>
      </section>
    </div>
  `
})
export class CatalogoPageComponent {
  readonly productoService = inject(ProductoService);

  agregarAlCarrito(producto: Producto): void {
    // Lo implementaremos en el Ejercicio 3
    console.log("Producto agregado:", producto.nombre);
  }
}
```

### ✅ Verificación del Ejercicio 2

- [ ] El catálogo muestra 8 productos en tarjetas visuales
- [ ] Filtrar por categoría (ej. "laptops") muestra solo 2 productos
- [ ] Escribir "4K" en el buscador muestra solo el monitor 4K
- [ ] El ordenamiento por precio funciona correctamente
- [ ] El bloque `@empty` se muestra cuando no hay resultados
- [ ] Los productos destacados muestran la etiqueta ⭐

### 🎯 Tarea para el lector
Añade un filtro de rango de precio (mín-máx) utilizando dos inputs numéricos y un nuevo `computed()` que combine todos los filtros. Verifica que los filtros se pueden combinar entre sí.

---

## Ejercicio 3: Carrito de Compras con NgRx Signals Store

### Objetivo
Implementar un almacén de estado global para el carrito de compras utilizando `@ngrx/signals`, con persistencia en `localStorage` y propiedades computadas.

### Paso 3.1: Instalar NgRx Signals

```bash
npm install @ngrx/signals
```

### Paso 3.2: Crear el CarritoStore

Crea el archivo `src/app/core/store/carrito.store.ts`:

```typescript
import { computed, inject } from "@angular/core";
import { signalStore, withState, withComputed, withMethods, withHooks, patchState } from "@ngrx/signals";
import { Producto, ItemCarrito } from "../models/producto.interface";

export interface CarritoState {
  items: ItemCarrito[];
  codigoDescuento: string;
}

const estadoInicial: CarritoState = {
  items: [],
  codigoDescuento: ""
};

export const CarritoStore = signalStore(
  { providedIn: "root" },

  withState(estadoInicial),

  withComputed(({ items, codigoDescuento }) => ({
    cantidadTotal: computed(() =>
      items().reduce((acc, item) => acc + item.cantidad, 0)
    ),
    subtotal: computed(() =>
      items().reduce((acc, item) => acc + item.producto.precio * item.cantidad, 0)
    ),
    porcentajeDescuento: computed(() => {
      const descuentos: Record<string, number> = {
        "TECHSTORE10": 0.10,
        "GDE_ANGULAR": 0.15,
        "VIP25": 0.25
      };
      return descuentos[codigoDescuento().toUpperCase()] ?? 0;
    }),
    totalFinal: computed(() => {
      const sub = items().reduce((acc, item) => acc + item.producto.precio * item.cantidad, 0);
      const descuentos: Record<string, number> = {
        "TECHSTORE10": 0.10, "GDE_ANGULAR": 0.15, "VIP25": 0.25
      };
      const desc = descuentos[codigoDescuento().toUpperCase()] ?? 0;
      return sub * (1 - desc);
    }),
    esVacio: computed(() => items().length === 0)
  })),

  withMethods((store) => ({
    agregar(producto: Producto): void {
      patchState(store, (state) => {
        const existente = state.items.find(i => i.producto.id === producto.id);
        if (existente) {
          return {
            items: state.items.map(i =>
              i.producto.id === producto.id
                ? { ...i, cantidad: Math.min(i.cantidad + 1, producto.stock) }
                : i
            )
          };
        }
        return { items: [...state.items, { producto, cantidad: 1 }] };
      });
    },

    decrementar(productoId: string): void {
      patchState(store, (state) => {
        const item = state.items.find(i => i.producto.id === productoId);
        if (item && item.cantidad <= 1) {
          return { items: state.items.filter(i => i.producto.id !== productoId) };
        }
        return {
          items: state.items.map(i =>
            i.producto.id === productoId ? { ...i, cantidad: i.cantidad - 1 } : i
          )
        };
      });
    },

    eliminar(productoId: string): void {
      patchState(store, (state) => ({
        items: state.items.filter(i => i.producto.id !== productoId)
      }));
    },

    aplicarDescuento(codigo: string): void {
      patchState(store, { codigoDescuento: codigo });
    },

    vaciar(): void {
      patchState(store, { items: [], codigoDescuento: "" });
    }
  })),

  withHooks({
    onInit(store) {
      // Restaurar estado desde localStorage
      const saved = localStorage.getItem("techstore-carrito");
      if (saved) {
        try {
          const parsed = JSON.parse(saved);
          patchState(store, { items: parsed.items ?? [] });
        } catch { /* Ignorar datos corruptos */ }
      }
    }
  })
);
```

### Paso 3.3: Conectar el catálogo con el carrito

Actualiza el método `agregarAlCarrito` en `CatalogoPageComponent`:

```typescript
import { CarritoStore } from "../../../core/store/carrito.store";

export class CatalogoPageComponent {
  readonly productoService = inject(ProductoService);
  private readonly carritoStore = inject(CarritoStore);

  agregarAlCarrito(producto: Producto): void {
    this.carritoStore.agregar(producto);
  }
}
```

### Paso 3.4: Construir la página del carrito

Edita `src/app/features/carrito/carrito-page/carrito-page.component.ts`:

```typescript
import { Component, inject, ChangeDetectionStrategy } from "@angular/core";
import { CurrencyPipe } from "@angular/common";
import { RouterLink } from "@angular/router";
import { CarritoStore } from "../../../core/store/carrito.store";

@Component({
  selector: "app-carrito-page",
  standalone: true,
  imports: [CurrencyPipe, RouterLink],
  changeDetection: ChangeDetectionStrategy.OnPush,
  template: `
    <h1 class="text-2xl font-extrabold text-gray-900 mb-6">Mi Carrito</h1>

    @if (carrito.esVacio()) {
      <div class="text-center py-16 bg-white rounded-lg shadow-sm">
        <p class="text-5xl mb-4">🛒</p>
        <p class="text-lg font-semibold text-gray-600">Tu carrito está vacío</p>
        <a routerLink="/catalogo" class="inline-block mt-4 px-6 py-2 bg-blue-600 text-white rounded-lg">
          Explorar catálogo
        </a>
      </div>
    } @else {
      <div class="grid grid-cols-1 lg:grid-cols-3 gap-6">
        <!-- Lista de items -->
        <div class="lg:col-span-2 space-y-3">
          @for (item of carrito.items(); track item.producto.id) {
            <div class="bg-white p-4 rounded-lg shadow-sm border flex items-center gap-4">
              <span class="text-3xl">🖥️</span>
              <div class="flex-1">
                <h3 class="font-bold text-sm">{{ item.producto.nombre }}</h3>
                <p class="text-blue-600 font-semibold">{{ item.producto.precio | currency:'EUR' }}</p>
              </div>
              <div class="flex items-center gap-2">
                <button (click)="carrito.decrementar(item.producto.id)"
                  class="w-8 h-8 rounded-full bg-gray-100 hover:bg-gray-200 text-lg font-bold">−</button>
                <span class="w-8 text-center font-semibold">{{ item.cantidad }}</span>
                <button (click)="carrito.agregar(item.producto)"
                  class="w-8 h-8 rounded-full bg-gray-100 hover:bg-gray-200 text-lg font-bold">+</button>
              </div>
              <button (click)="carrito.eliminar(item.producto.id)"
                class="text-red-500 hover:text-red-700 text-sm font-semibold">Eliminar</button>
            </div>
          }
        </div>

        <!-- Resumen del pedido -->
        <div class="bg-white p-6 rounded-lg shadow-sm border h-fit sticky top-4">
          <h2 class="font-bold text-lg mb-4">Resumen del Pedido</h2>
          <div class="space-y-2 text-sm">
            <div class="flex justify-between">
              <span>Productos ({{ carrito.cantidadTotal() }})</span>
              <span>{{ carrito.subtotal() | currency:'EUR' }}</span>
            </div>
            @if (carrito.porcentajeDescuento() > 0) {
              <div class="flex justify-between text-green-600">
                <span>Descuento ({{ carrito.porcentajeDescuento() * 100 }}%)</span>
                <span>-{{ carrito.subtotal() * carrito.porcentajeDescuento() | currency:'EUR' }}</span>
              </div>
            }
            <hr>
            <div class="flex justify-between font-bold text-lg">
              <span>Total</span>
              <span class="text-blue-600">{{ carrito.totalFinal() | currency:'EUR' }}</span>
            </div>
          </div>

          <!-- Código de descuento -->
          <div class="mt-4">
            <input
              #codigoInput
              type="text"
              placeholder="Código de descuento"
              class="w-full border rounded px-3 py-2 text-sm mb-2">
            <button
              (click)="carrito.aplicarDescuento(codigoInput.value)"
              class="w-full bg-gray-100 hover:bg-gray-200 py-2 rounded text-sm font-semibold">
              Aplicar código
            </button>
          </div>

          <a routerLink="/checkout"
            class="block w-full mt-4 bg-blue-600 text-white text-center py-3 rounded-lg font-bold hover:bg-blue-700">
            Proceder al Checkout →
          </a>
        </div>
      </div>
    }
  `
})
export class CarritoPageComponent {
  readonly carrito = inject(CarritoStore);
}
```

### ✅ Verificación del Ejercicio 3

- [ ] Agregar productos desde el catálogo se refleja inmediatamente en el carrito
- [ ] Los botones +/− actualizan la cantidad correctamente
- [ ] Eliminar un producto lo remueve del carrito
- [ ] El código "TECHSTORE10" aplica un 10% de descuento visible en el resumen
- [ ] Carrito vacío muestra el estado vacío con enlace al catálogo

### 🎯 Tarea para el lector
Implementa la persistencia del carrito usando un `effect()` que guarde en `localStorage` cada vez que `items()` cambie. Verifica que al recargar la página, el carrito conserve su contenido.

---

## Ejercicio 4: Formulario de Checkout con Validaciones Avanzadas

### Objetivo
Construir un formulario reactivo tipado con validaciones síncronas personalizadas, feedback visual de errores y protección contra envío inválido.

### Paso 4.1: Implementar el formulario de checkout

Edita `src/app/features/checkout/checkout-page/checkout-page.component.ts`:

```typescript
import { Component, inject, ChangeDetectionStrategy, signal } from "@angular/core";
import { NonNullableFormBuilder, ReactiveFormsModule, Validators, AbstractControl, ValidationErrors } from "@angular/forms";
import { CurrencyPipe } from "@angular/common";
import { Router } from "@angular/router";
import { CarritoStore } from "../../../core/store/carrito.store";

// Validador personalizado: número de tarjeta con algoritmo de Luhn simplificado
function tarjetaValidaValidator(control: AbstractControl): ValidationErrors | null {
  const valor = (control.value || "").replace(/\s/g, "");
  if (!valor) return null;
  if (!/^\d{16}$/.test(valor)) return { tarjetaInvalida: { mensaje: "Debe tener exactamente 16 dígitos" } };
  return null;
}

// Validador personalizado: fecha de expiración futura
function fechaExpiracionValidator(control: AbstractControl): ValidationErrors | null {
  const valor = control.value || "";
  if (!valor) return null;
  const match = valor.match(/^(\d{2})\/(\d{2})$/);
  if (!match) return { formatoInvalido: true };
  const mes = parseInt(match[1], 10);
  const anio = parseInt("20" + match[2], 10);
  const ahora = new Date();
  if (mes < 1 || mes > 12) return { mesInvalido: true };
  if (anio < ahora.getFullYear() || (anio === ahora.getFullYear() && mes < ahora.getMonth() + 1)) {
    return { tarjetaExpirada: true };
  }
  return null;
}

@Component({
  selector: "app-checkout-page",
  standalone: true,
  imports: [ReactiveFormsModule, CurrencyPipe],
  changeDetection: ChangeDetectionStrategy.OnPush,
  template: `
    <h1 class="text-2xl font-extrabold text-gray-900 mb-6">Finalizar Compra</h1>

    @if (compraCompletada()) {
      <div class="text-center py-16 bg-white rounded-lg shadow-sm">
        <p class="text-5xl mb-4">✅</p>
        <h2 class="text-xl font-bold text-green-600">¡Compra realizada con éxito!</h2>
        <p class="text-gray-500 mt-2">Recibirás un correo de confirmación en breve.</p>
      </div>
    } @else {
      <form [formGroup]="checkoutForm" (ngSubmit)="procesarCompra()" class="max-w-2xl mx-auto space-y-6">
        
        <!-- Datos personales -->
        <fieldset class="bg-white p-6 rounded-lg shadow-sm border">
          <legend class="text-lg font-bold text-gray-900 mb-4">Datos Personales</legend>
          <div class="grid grid-cols-1 md:grid-cols-2 gap-4">
            <div>
              <label for="nombre" class="block text-sm font-semibold mb-1">Nombre completo *</label>
              <input id="nombre" formControlName="nombre" class="w-full border rounded px-3 py-2">
              @if (checkoutForm.controls.nombre.touched && checkoutForm.controls.nombre.hasError('required')) {
                <p class="text-red-500 text-xs mt-1">El nombre es obligatorio</p>
              }
              @if (checkoutForm.controls.nombre.hasError('minlength')) {
                <p class="text-red-500 text-xs mt-1">Mínimo 3 caracteres</p>
              }
            </div>
            <div>
              <label for="email" class="block text-sm font-semibold mb-1">Email *</label>
              <input id="email" formControlName="email" type="email" class="w-full border rounded px-3 py-2">
              @if (checkoutForm.controls.email.touched && checkoutForm.controls.email.hasError('email')) {
                <p class="text-red-500 text-xs mt-1">Email inválido</p>
              }
            </div>
          </div>
          <div class="mt-4">
            <label for="direccion" class="block text-sm font-semibold mb-1">Dirección de envío *</label>
            <input id="direccion" formControlName="direccion" class="w-full border rounded px-3 py-2">
          </div>
        </fieldset>

        <!-- Datos de pago -->
        <fieldset class="bg-white p-6 rounded-lg shadow-sm border">
          <legend class="text-lg font-bold text-gray-900 mb-4">Método de Pago</legend>
          <div class="space-y-4">
            <div>
              <label for="tarjeta" class="block text-sm font-semibold mb-1">Número de tarjeta *</label>
              <input id="tarjeta" formControlName="numeroTarjeta" placeholder="1234 5678 9012 3456"
                class="w-full border rounded px-3 py-2 font-mono tracking-widest">
              @if (checkoutForm.controls.numeroTarjeta.touched && checkoutForm.controls.numeroTarjeta.hasError('tarjetaInvalida')) {
                <p class="text-red-500 text-xs mt-1">
                  {{ checkoutForm.controls.numeroTarjeta.getError('tarjetaInvalida').mensaje }}
                </p>
              }
            </div>
            <div class="grid grid-cols-2 gap-4">
              <div>
                <label for="expiracion" class="block text-sm font-semibold mb-1">Expiración *</label>
                <input id="expiracion" formControlName="expiracion" placeholder="MM/AA"
                  class="w-full border rounded px-3 py-2">
                @if (checkoutForm.controls.expiracion.touched && checkoutForm.controls.expiracion.hasError('tarjetaExpirada')) {
                  <p class="text-red-500 text-xs mt-1">La tarjeta ha expirado</p>
                }
              </div>
              <div>
                <label for="cvv" class="block text-sm font-semibold mb-1">CVV *</label>
                <input id="cvv" formControlName="cvv" maxlength="4" class="w-full border rounded px-3 py-2">
              </div>
            </div>
          </div>
        </fieldset>

        <!-- Resumen y envío -->
        <div class="bg-blue-50 p-6 rounded-lg border border-blue-200">
          <div class="flex justify-between items-center">
            <div>
              <p class="font-bold text-lg">Total a pagar:</p>
              <p class="text-2xl font-extrabold text-blue-600">{{ carrito.totalFinal() | currency:'EUR' }}</p>
            </div>
            <button type="submit" [disabled]="checkoutForm.invalid"
              class="px-8 py-3 bg-blue-600 text-white font-bold rounded-lg hover:bg-blue-700 
                     disabled:bg-gray-300 disabled:cursor-not-allowed transition-colors">
              Confirmar Compra
            </button>
          </div>
        </div>
      </form>
    }
  `
})
export class CheckoutPageComponent {
  private readonly fb = inject(NonNullableFormBuilder);
  private readonly router = inject(Router);
  readonly carrito = inject(CarritoStore);
  readonly compraCompletada = signal(false);

  readonly checkoutForm = this.fb.group({
    nombre: ["", [Validators.required, Validators.minLength(3)]],
    email: ["", [Validators.required, Validators.email]],
    direccion: ["", [Validators.required, Validators.minLength(10)]],
    numeroTarjeta: ["", [Validators.required, tarjetaValidaValidator]],
    expiracion: ["", [Validators.required, fechaExpiracionValidator]],
    cvv: ["", [Validators.required, Validators.pattern(/^\d{3,4}$/)]]
  });

  procesarCompra(): void {
    if (this.checkoutForm.invalid) {
      this.checkoutForm.markAllAsTouched();
      return;
    }
    
    const datos = this.checkoutForm.getRawValue();
    console.log("Pedido procesado:", {
      cliente: datos.nombre,
      email: datos.email,
      items: this.carrito.items().length,
      total: this.carrito.totalFinal()
    });
    
    this.carrito.vaciar();
    this.compraCompletada.set(true);
  }
}
```

### ✅ Verificación del Ejercicio 4

- [ ] El formulario no se envía si hay campos vacíos
- [ ] Errores de validación se muestran al tocar un campo y perder el foco
- [ ] El validador de tarjeta rechaza valores con menos de 16 dígitos
- [ ] El validador de expiración rechaza fechas pasadas
- [ ] Al confirmar la compra, el carrito se vacía y aparece el mensaje de éxito

### 🎯 Tarea para el lector
Implementa un **validador asíncrono** que simule verificar si el email ya está registrado en la base de datos (usa un `setTimeout` de 1 segundo para simular la latencia). Muestra un spinner de carga al lado del input mientras la validación está en curso usando `checkoutForm.controls.email.pending`.

---

## Ejercicio 5: Optimización con @defer y Análisis de Bundle

### Objetivo
Aplicar carga diferida declarativa a componentes pesados del catálogo y medir el impacto real en el tamaño del bundle.

### Paso 5.1: Crear un componente "pesado" de reseñas

Crea `src/app/features/catalogo/components/resenas-producto/resenas-producto.component.ts`:

```typescript
import { Component, input, signal, OnInit, ChangeDetectionStrategy } from "@angular/core";

interface Resena {
  autor: string;
  puntuacion: number;
  texto: string;
  fecha: string;
}

@Component({
  selector: "app-resenas-producto",
  standalone: true,
  changeDetection: ChangeDetectionStrategy.OnPush,
  template: `
    <div class="mt-4 p-4 bg-gray-50 rounded-lg border">
      <h4 class="font-bold text-sm mb-3">Opiniones de clientes</h4>
      @for (resena of resenas(); track resena.autor) {
        <div class="mb-3 pb-3 border-b last:border-0">
          <div class="flex items-center gap-2">
            <span class="text-xs font-semibold">{{ resena.autor }}</span>
            <span class="text-yellow-500 text-xs">
              @for (star of getEstrellas(resena.puntuacion); track $index) { ⭐ }
            </span>
          </div>
          <p class="text-xs text-gray-600 mt-1">{{ resena.texto }}</p>
        </div>
      }
    </div>
  `
})
export class ResenasProductoComponent implements OnInit {
  readonly productoId = input.required<string>();
  readonly resenas = signal<Resena[]>([]);

  ngOnInit(): void {
    // Simular carga de reseñas
    this.resenas.set([
      { autor: "María G.", puntuacion: 5, texto: "Excelente producto, superó mis expectativas.", fecha: "2025-03-15" },
      { autor: "Carlos R.", puntuacion: 4, texto: "Muy bueno, la relación calidad-precio es inmejorable.", fecha: "2025-02-20" },
      { autor: "Laura M.", puntuacion: 5, texto: "Lo recomiendo al 100%. Entrega rápida.", fecha: "2025-01-10" }
    ]);
  }

  getEstrellas(n: number): number[] {
    return Array(n).fill(0);
  }
}
```

### Paso 5.2: Aplicar @defer en la tarjeta de producto

Actualiza `tarjeta-producto.component.ts` añadiendo las reseñas con carga diferida:

```html
<!-- Añadir al final de la tarjeta, antes del cierre de </article> -->
@defer (on interaction; prefetch on hover) {
  <app-resenas-producto [productoId]="producto().id" />
} @placeholder {
  <button class="w-full mt-2 text-xs text-blue-500 hover:text-blue-700 py-2 text-center">
    Ver opiniones de clientes →
  </button>
} @loading (after 100ms; minimum 300ms) {
  <div class="text-center py-3">
    <span class="text-xs text-gray-400 animate-pulse">Cargando opiniones...</span>
  </div>
}
```

### Paso 5.3: Medir el impacto en el bundle

```bash
# Compilar para producción
ng build --configuration production

# Instalar herramienta de análisis
npm install -D source-map-explorer

# Analizar el bundle principal
npx source-map-explorer dist/techstore/browser/*.js
```

### ✅ Verificación del Ejercicio 5

- [ ] Las reseñas NO se cargan al abrir la página del catálogo
- [ ] Al pasar el ratón por una tarjeta, el chunk se descarga (pestaña Network de DevTools)
- [ ] Al hacer clic en "Ver opiniones", las reseñas se renderizan instantáneamente
- [ ] El bundle principal (`main.js`) no contiene el código de `ResenasProductoComponent`

---

## Ejercicio 6: Escritura de Tests Unitarios y E2E

### Objetivo
Escribir tests unitarios para el `CarritoStore` y un test E2E que verifique el flujo completo de compra.

### Paso 6.1: Test unitario del CarritoStore

Crea `src/app/core/store/carrito.store.spec.ts`:

```typescript
import { TestBed } from "@angular/core/testing";
import { CarritoStore } from "./carrito.store";
import { Producto } from "../models/producto.interface";
import { describe, beforeEach, it, expect } from "vitest";

describe("CarritoStore", () => {
  let store: InstanceType<typeof CarritoStore>;

  const productoMock: Producto = {
    id: "TEST-001", nombre: "Producto Test", descripcion: "Desc",
    precio: 100, categoria: "laptops", imagenUrl: "", stock: 10, destacado: false
  };

  beforeEach(() => {
    TestBed.configureTestingModule({ providers: [CarritoStore] });
    store = TestBed.inject(CarritoStore);
    localStorage.clear();
  });

  it("debe iniciar con carrito vacío", () => {
    expect(store.items()).toEqual([]);
    expect(store.cantidadTotal()).toBe(0);
    expect(store.esVacio()).toBe(true);
  });

  it("debe agregar un producto nuevo con cantidad 1", () => {
    store.agregar(productoMock);
    expect(store.items().length).toBe(1);
    expect(store.items()[0].cantidad).toBe(1);
    expect(store.cantidadTotal()).toBe(1);
    expect(store.subtotal()).toBe(100);
  });

  it("debe incrementar la cantidad si el producto ya existe", () => {
    store.agregar(productoMock);
    store.agregar(productoMock);
    expect(store.items().length).toBe(1);
    expect(store.items()[0].cantidad).toBe(2);
    expect(store.subtotal()).toBe(200);
  });

  it("debe aplicar el descuento TECHSTORE10 correctamente", () => {
    store.agregar(productoMock);
    store.aplicarDescuento("TECHSTORE10");
    expect(store.porcentajeDescuento()).toBe(0.10);
    expect(store.totalFinal()).toBe(90);
  });

  it("debe vaciar completamente el carrito", () => {
    store.agregar(productoMock);
    store.vaciar();
    expect(store.esVacio()).toBe(true);
    expect(store.subtotal()).toBe(0);
  });
});
```

### Paso 6.2: Test E2E con Playwright

Crea `e2e/flujo-compra.spec.ts`:

```typescript
import { test, expect } from "@playwright/test";

test.describe("Flujo de compra completo", () => {
  test("debe permitir agregar un producto, ir al carrito y completar el checkout", async ({ page }) => {
    // 1. Navegar al catálogo
    await page.goto("http://localhost:4200/catalogo");
    await expect(page.locator("h1")).toContainText("Catálogo");

    // 2. Agregar el primer producto al carrito
    const primerBotonComprar = page.locator("button:has-text('Agregar')").first();
    await primerBotonComprar.click();

    // 3. Navegar al carrito
    await page.goto("http://localhost:4200/carrito");
    await expect(page.locator("h1")).toContainText("Mi Carrito");

    // 4. Verificar que hay al menos un item
    const items = page.locator("[class*='shadow-sm border']");
    await expect(items.first()).toBeVisible();

    // 5. Ir al checkout
    await page.click("text=Proceder al Checkout");
    await expect(page).toHaveURL(/checkout/);

    // 6. Rellenar formulario
    await page.fill("#nombre", "Juan Developer");
    await page.fill("#email", "juan@techstore.com");
    await page.fill("#direccion", "Calle Angular 42, Madrid, España");
    await page.fill("#tarjeta", "4111111111111111");
    await page.fill("#expiracion", "12/28");
    await page.fill("#cvv", "123");

    // 7. Confirmar compra
    await page.click("text=Confirmar Compra");
    
    // 8. Verificar éxito
    await expect(page.locator("text=¡Compra realizada con éxito!")).toBeVisible();
  });
});
```

### ✅ Verificación del Ejercicio 6

- [ ] Todos los tests unitarios pasan: `npx vitest run`
- [ ] El test E2E pasa: `npx playwright test`

---

## Ejercicio 7: SSR, SEO y Despliegue a Producción

### Objetivo
Activar SSR, configurar metadatos SEO dinámicos y compilar para producción.

### Paso 7.1: Activar SSR

```bash
ng add @angular/ssr
```

### Paso 7.2: Proteger accesos al navegador

Revisa todos los usos de `localStorage` en el `CarritoStore` y envuélvelos en comprobaciones de plataforma:

```typescript
import { PLATFORM_ID, inject } from "@angular/core";
import { isPlatformBrowser } from "@angular/common";

// Dentro de withHooks:
onInit(store) {
  const platformId = inject(PLATFORM_ID);
  if (isPlatformBrowser(platformId)) {
    const saved = localStorage.getItem("techstore-carrito");
    // ...
  }
}
```

### Paso 7.3: Compilar y verificar

```bash
ng build
npm run serve:ssr:techstore
```

### ✅ Verificación del Ejercicio 7

- [ ] La aplicación compila sin errores en modo SSR
- [ ] El HTML fuente (View Page Source) contiene el contenido del catálogo pre-renderizado
- [ ] No hay errores de `window is not defined` o `localStorage is not defined`

---

## Retos Adicionales para el Lector

Si has completado todos los ejercicios, aquí tienes retos avanzados para seguir practicando:

1. **🔥 Wishlist**: Implementa un sistema de lista de deseos utilizando un segundo `signalStore` que persista en `localStorage`.

2. **🔍 Búsqueda con debounce**: Conecta el filtro de búsqueda a `toObservable()` y aplica `debounceTime(300)` para evitar cálculos en cada pulsación de tecla.

3. **📊 Historial de pedidos**: Crea una página `/pedidos` que muestre los pedidos completados almacenados en `localStorage`.

4. **🎨 Tema oscuro**: Implementa un toggle de tema oscuro/claro usando un Signal global y `effect()` que aplique el atributo `data-theme` al `<html>`.

5. **♿ Accesibilidad**: Añade `aria-label`, `role` y `aria-live="polite"` para anunciar cuando se agrega un producto al carrito.

6. **📱 PWA**: Convierte la app en PWA con `ng add @angular/pwa` y configura la estrategia de caché para el catálogo.

---

## Resumen del Capítulo

En este capítulo has construido, paso a paso, una aplicación E-Commerce funcional y completa que integra:

* **Signals y `computed()`** para filtros reactivos de catálogo sin suscripciones manuales
* **NgRx Signals Store** con `withState`, `withComputed`, `withMethods` y `withHooks` para estado global del carrito
* **Formularios Reactivos tipados** con `NonNullableFormBuilder` y validadores personalizados
* **`@defer`** con `prefetch on hover` y `on interaction` para optimización de bundle
* **Tests unitarios** del Store y **tests E2E** con Playwright
* **SSR** con protección de APIs del navegador

Cada ejercicio ha sido diseñado para reforzar los conceptos de capítulos anteriores en un contexto práctico real. Los retos adicionales te permiten seguir explorando y profundizando en las áreas que más te interesen.
