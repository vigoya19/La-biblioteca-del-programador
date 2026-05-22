# Capítulo 4: Angular Signals — El Nuevo Motor Reactivo

> "Signals no es solo una API nueva. Es la reescritura del contrato fundamental entre Angular y el desarrollador sobre cómo y cuándo se actualiza la interfaz de usuario."

Durante más de 7 años (desde Angular 2 en 2016), la detección de cambios de Angular ha funcionado con un modelo conceptualmente simple pero computacionalmente brutal: **Zone.js** intercepta todas las operaciones asíncronas del navegador (eventos del DOM, timers, peticiones HTTP, promesas) y, cuando cualquiera de ellas se completa, Angular ejecuta un barrido completo del árbol de componentes desde la raíz, comparando cada expresión de cada plantilla para determinar si el DOM necesita actualizarse.

Este modelo ("dirty checking global") funciona sin que el desarrollador piense en reactividad, pero introduce problemas severos en aplicaciones a escala:

* **Ineficiencia computacional**: Una aplicación con 500 componentes ejecuta hasta 500 ciclos de comprobación por cada click del usuario, la mayoría de los cuales no encuentran cambios.
* **Overhead de Zone.js**: Esta librería pesa ~15-30KB gzip, modifica las APIs nativas del navegador (monkey-patching) y contamina los stack traces de errores, haciéndolos ilegibles.
* **Falta de granularidad**: Angular no sabe QUÉ cambió; solo sabe que ALGO cambió en algún lugar.

**Angular Signals** resuelve estos problemas implementando un modelo de **reactividad fina y granular (Fine-Grained Reactivity)** inspirado en frameworks como SolidJS. En lugar de barrer todo el árbol, cada Signal notifica exactamente a las partes de la UI que dependen de él, permitiendo actualizaciones DOM quirúrgicas y abriendo la puerta a un Angular completamente libre de Zone.js.

---

## 4.1 La Teoría de la Reactividad Fina: El Grafo Push/Pull

Para entender Signals en profundidad, necesitamos comprender el modelo computacional que los sustenta: el **grafo de dependencias reactivas push/pull**.

> [!NOTE]
> ### 📊 La Analogía de la Hoja de Cálculo Inteligente (Excel)
> 
> Para entender el motor de reactividad de los **Signals**, imagina que estás trabajando en una hoja de cálculo de Excel:
> 
> - En la celda **A1** escribes el valor `100` (este es tu **Writable Signal**: la fuente original del dato).
> - En la celda **B1** escribes la fórmula `=A1 * 2` (este es tu **Computed Signal**: un dato que se calcula a partir de otro).
> - En la celda **C1** escribes la fórmula `=B1 - 10` (este es tu **Consumidor**: la plantilla de tu página web que muestra el precio final).
> 
> Angular Signals gestiona esta conexión en dos fases ultrarrápidas y perezosas:
> 
> 1. **Fase PUSH (La Alarma)**: Si cambias el valor de **A1** a `200`, Excel no calcula inmediatamente los valores de B1 y C1. En su lugar, simplemente envía un "grito de alarma" a través del cableado: *"¡Oigan, A1 ha cambiado! Por lo tanto, el valor que tengan guardado actualmente en B1 y C1 ya no es confiable, márquense como desactualizados (dirty)"*. Esto toma apenas una milésima de segundo.
> 2. **Fase PULL (El Recálculo Perezoso)**: Solo cuando el usuario dirige la mirada a la celda **C1** (es decir, cuando el navegador necesita renderizar la pantalla), C1 "jala" del sistema: *"Oye B1, dame tu valor actualizado"*. B1 a su vez le pide el dato a A1, calcula `200 * 2 = 400`, se lo pasa a C1, y C1 calcula el resultado final de `390`.
> 
> **¿Por qué es genial?** Si cambias el valor de A1 100 veces seguidas pero nadie está mirando la celda C1, Angular **cero veces** ejecutará la multiplicación o renderizado. Ahorro total de batería y procesador.

### El Modelo Mental

Imagina tu aplicación como un grafo dirigido acíclico (DAG) donde:
* Los **nodos productores** (Writable Signals) son las fuentes de verdad del estado.
* Los **nodos derivados** (Computed Signals) son cálculos que dependen de productores.
* Los **nodos consumidores** (Templates, Effects) son los puntos finales que leen valores y producen efectos secundarios.

```
Productores                 Derivados                    Consumidores
(Fuentes de Estado)         (Cálculos Memorizados)       (Efectos Secundarios)

┌──────────────┐
│ precio()     │──────┐
│ signal(100)  │      ├──────► ┌─────────────────┐
└──────────────┘      │        │ subtotal()       │         ┌──────────────────┐
                      │        │ computed(         │────────►│ Template HTML    │
┌──────────────┐      │        │   precio*cantidad)│         │ {{ subtotal() }} │
│ cantidad()   │──────┘        └─────────────────┘         └──────────────────┘
│ signal(3)    │                        │
└──────────────┘                        │
                                        │
┌──────────────┐      ┌────────────────▼┐         ┌──────────────────┐
│ descuento()  │──────►│ totalFinal()    │────────►│ effect()         │
│ signal(0.1)  │       │ computed(       │         │ guardar en       │
└──────────────┘       │  subtotal*desc) │         │ localStorage     │
                       └─────────────────┘         └──────────────────┘
```

### El Protocolo Push/Pull

El modelo reactivo de Angular Signals utiliza un protocolo **push/pull híbrido**:

1. **Push (Notificación de invalidación)**: Cuando un Writable Signal cambia su valor, empuja una señal de "invalidación" a todos sus dependientes directos. Estos dependientes propagan la invalidación hacia arriba hasta los consumidores finales. **Importante**: en esta fase NO se recalcula ningún valor. Solo se marca como "posiblemente sucio" (dirty).

2. **Pull (Recálculo perezoso)**: Cuando un consumidor (la plantilla o un effect) necesita el valor actualizado, "tira" del grafo recalculando solo los nodos que están marcados como dirty. Si un computed descubre que su valor calculado no ha cambiado realmente (gracias a la comparación de igualdad), detiene la propagación.

```
Fase PUSH (Instantánea, O(1) por dependiente):
precio.set(200)
  → marca subtotal como "dirty"
    → marca totalFinal como "dirty"
      → marca template como "necesita actualización"
      → marca effect como "necesita re-ejecución"

Fase PULL (Perezosa, solo cuando se lee):
template lee totalFinal()
  → totalFinal está dirty, recalcula
    → necesita subtotal(), que está dirty, recalcula
      → necesita precio() (200) y cantidad() (3)
      → subtotal = 200 * 3 = 600
    → necesita descuento() (0.1)
    → totalFinal = 600 * (1 - 0.1) = 540
  → template actualiza el DOM con "540"
```

Esta arquitectura garantiza que:
* Los valores derivados NUNCA se recalculan innecesariamente (evaluación perezosa).
* Los consumidores SOLO se actualizan cuando el valor final ha cambiado realmente (glitch-free).
* No hay "diamond problems" ni actualizaciones duplicadas.

---

## 4.2 Writable Signals: Contenedores Reactivos de Estado

Un **Writable Signal** es un contenedor reactivo de lectura/escritura que envuelve un valor y notifica a sus dependientes cuando ese valor cambia.

### Creación y Lectura

```typescript
import { signal } from "@angular/core";

// Signal de tipo primitivo
const contador = signal<number>(0);

// Signal de tipo complejo
const usuario = signal<{ nombre: string; edad: number }>({
  nombre: "Andrés",
  edad: 28
});

// Lectura: invocar el signal como función
console.log(contador());      // 0
console.log(usuario().nombre); // "Andrés"
```

### Métodos de Actualización

```typescript
// .set(): Reemplaza el valor completo
contador.set(10);

// .update(): Actualiza basándose en el valor anterior
contador.update(prev => prev + 1);

// .update() con objetos (SIEMPRE inmutabilidad)
usuario.update(prev => ({ ...prev, edad: prev.edad + 1 }));

// NUNCA hacer esto (mutación directa, NO dispara actualizaciones):
// usuario().edad = 29;  ← INCORRECTO: el Signal no detecta este cambio
```

### Función de Igualdad Personalizada (`equal`)

Por defecto, Angular usa `Object.is()` para comparar el valor anterior y el nuevo. Si son iguales, no notifica a los dependientes. Puedes personalizar esta comparación para objetos complejos:

```typescript
interface Coordenada {
  lat: number;
  lng: number;
}

// Sin función de igualdad: cada .set() notifica aunque lat/lng sean iguales
const posicionIngenua = signal<Coordenada>({ lat: 40.4168, lng: -3.7038 });

// Con función de igualdad: solo notifica si lat o lng realmente cambiaron
const posicion = signal<Coordenada>(
  { lat: 40.4168, lng: -3.7038 },
  {
    equal: (a, b) => a.lat === b.lat && a.lng === b.lng
  }
);

// Esto NO disparará actualizaciones porque lat y lng son iguales:
posicion.set({ lat: 40.4168, lng: -3.7038 });

// Esto SÍ disparará porque lng cambió:
posicion.set({ lat: 40.4168, lng: -3.7040 });
```

### `asReadonly()`: Proteger el Estado

Para exponer un Signal de solo lectura a consumidores externos (otros componentes, plantillas), usamos `.asReadonly()`:

```typescript
@Injectable({ providedIn: "root" })
export class AuthService {
  // Estado privado modificable
  private readonly _usuario = signal<Usuario | null>(null);
  
  // Exposición pública de solo lectura (el consumidor NO puede llamar .set() o .update())
  readonly usuario = this._usuario.asReadonly();
  
  login(datos: Credenciales): void {
    // Solo el servicio puede modificar el estado
    this._usuario.set({ nombre: datos.email, rol: "admin" });
  }
}
```

---

## 4.3 Computed Signals: Derivaciones Memorizadas

Un **Computed Signal** es un Signal de solo lectura cuyo valor se deriva de otros Signals. Se recalcula de forma **perezosa** (lazy) y se **memoriza** (cachea) hasta que alguna de sus dependencias cambie.

```typescript
import { signal, computed } from "@angular/core";

const precioUnitario = signal(100);
const cantidad = signal(3);
const descuento = signal(0.1);

// Computed derivado de dos signals
const subtotal = computed(() => precioUnitario() * cantidad());

// Computed derivado de otro computed
const totalConDescuento = computed(() => subtotal() * (1 - descuento()));

console.log(totalConDescuento()); // 270

// Cambiar una dependencia invalida el computed
cantidad.set(5);
console.log(totalConDescuento()); // 450
```

### Propiedades Clave de `computed()`

1. **Evaluación perezosa**: El cuerpo de la función se ejecuta SOLO cuando alguien lee el valor del computed. Si nadie lee `totalConDescuento()`, la multiplicación nunca se ejecuta.

2. **Memorización automática**: Si lees `totalConDescuento()` 100 veces seguidas sin que cambie ninguna dependencia, el resultado se retorna de caché sin re-ejecutar la función. El cómputo se ejecuta exactamente una vez.

3. **Rastreo dinámico de dependencias**: Angular rastrea qué Signals se leen durante la ejecución del computed. Si la lógica tiene ramas condicionales, las dependencias pueden cambiar dinámicamente:

```typescript
const mostrarPrecioConIVA = signal(true);
const precioBase = signal(100);
const iva = signal(21);

const precioMostrado = computed(() => {
  if (mostrarPrecioConIVA()) {
    // En esta rama, depende de precioBase Y de iva
    return precioBase() * (1 + iva() / 100);
  }
  // En esta rama, SOLO depende de precioBase (deja de escuchar 'iva')
  return precioBase();
});
```

Cuando `mostrarPrecioConIVA` cambia a `false`, el computed deja de rastrear `iva`. Un cambio posterior en `iva` **no** disparará recálculo del computed. Esto es una optimización clave.

4. **Igualdad personalizada en computeds**: Al igual que los writable signals, los computeds aceptan una función `equal`:

```typescript
const usuariosFiltrados = computed(
  () => todosLosUsuarios().filter(u => u.activo),
  { equal: (prev, curr) => prev.length === curr.length && prev.every((u, i) => u.id === curr[i].id) }
);
```

---

## 4.4 Effects: Efectos Secundarios Reactivos

Un **Effect** es una operación que se ejecuta automáticamente cuando cambian los Signals leídos en su cuerpo. A diferencia de los computeds (que son derivaciones puras de datos), los effects producen **efectos secundarios**: guardar en localStorage, logging, sincronizar con APIs externas, manipular el DOM imperativo.

```typescript
import { Component, signal, effect } from "@angular/core";

@Component({
  selector: "app-configuracion",
  standalone: true,
  template: `
    <select (change)="cambiarTema($event)">
      <option value="light">Claro</option>
      <option value="dark">Oscuro</option>
    </select>
  `
})
export class ConfiguracionComponent {
  readonly tema = signal<"light" | "dark">("light");
  
  constructor() {
    // Effect que sincroniza el tema con localStorage y con el DOM
    effect(() => {
      const temaActual = this.tema();
      
      // Efecto secundario 1: Persistir en localStorage
      localStorage.setItem("tema-preferido", temaActual);
      
      // Efecto secundario 2: Modificar el atributo del <html> para CSS
      document.documentElement.setAttribute("data-theme", temaActual);
      
      console.log(`[CONFIG] Tema cambiado a: ${temaActual}`);
    });
  }

  cambiarTema(event: Event): void {
    const select = event.target as HTMLSelectElement;
    this.tema.set(select.value as "light" | "dark");
  }
}
```

### `onCleanup`: Limpieza de Recursos entre Ejecuciones

Cada vez que un effect se re-ejecuta (porque una dependencia cambió), puede necesitar limpiar recursos de la ejecución anterior: cerrar WebSockets, cancelar peticiones HTTP, remover event listeners. Para esto existe el callback `onCleanup`:

```typescript
effect((onCleanup) => {
  const userId = usuarioActivoId();
  
  // Crear un AbortController para cancelar peticiones si el userId cambia
  const controller = new AbortController();
  
  // Iniciar petición HTTP con signal de cancelación nativo
  fetch(`/api/perfil/${userId}`, { signal: controller.signal })
    .then(res => res.json())
    .then(data => perfilCompleto.set(data))
    .catch(err => {
      if (err.name !== "AbortError") {
        console.error("Error cargando perfil:", err);
      }
    });
  
  // Registrar limpieza: se ejecuta ANTES de la próxima ejecución del effect
  // o cuando el componente se destruye
  onCleanup(() => {
    controller.abort();
    console.log(`Petición cancelada para usuario: ${userId}`);
  });
});
```

### `untracked()`: Leer Signals sin Crear Dependencias

A veces necesitas leer un Signal dentro de un effect pero NO quieres que el effect se re-ejecute cuando ese Signal cambie. Para esto existe `untracked()`:

```typescript
import { signal, effect, untracked } from "@angular/core";

const contador = signal(0);
const multiplicador = signal(2);

effect(() => {
  // El effect se re-ejecuta cuando 'contador' cambia
  const valor = contador();
  
  // Pero NO se re-ejecuta cuando 'multiplicador' cambia
  const mult = untracked(() => multiplicador());
  
  console.log(`Resultado: ${valor * mult}`);
});

contador.set(5);      // Effect se ejecuta: "Resultado: 10"
multiplicador.set(3); // Effect NO se ejecuta (multiplicador es untracked)
contador.set(6);      // Effect se ejecuta: "Resultado: 18" (usa el multiplicador actual = 3)
```

### Reglas de Oro para Effects

1. **Nunca escribas en Signals dentro de un effect** (por defecto está prohibido y lanza error). Si necesitas derivar un valor, usa `computed()`. Si es absolutamente necesario, usa `allowSignalWrites: true` como último recurso.

2. **Los effects se ejecutan de forma asíncrona** como microtasks, no de forma síncrona inmediatamente después del `set()`. Esto permite que Angular agrupe múltiples cambios de estado antes de ejecutar effects.

3. **Los effects se destruyen automáticamente** con el contexto de inyección que los creó (típicamente el componente). No necesitas desuscribirte manualmente.

---

## 4.5 Signal Queries: viewChild(), viewChildren(), contentChild(), contentChildren()

Angular v17.2+ introduce Signal Queries que reemplazan los decoradores clásicos `@ViewChild`, `@ViewChildren`, `@ContentChild` y `@ContentChildren`. Estas APIs retornan Signals en lugar de propiedades imperativas, integrándose perfectamente con `computed()` y `effect()`.

```typescript
import { Component, viewChild, viewChildren, contentChild, contentChildren, ElementRef, computed } from "@angular/core";

@Component({
  selector: "app-formulario-dinamico",
  standalone: true,
  template: `
    <form #formularioRef>
      <input #campoNombre type="text">
      <input #campoEmail type="email">
      <ng-content></ng-content>
    </form>
  `
})
export class FormularioDinamicoComponent {
  // Signal query: un solo elemento por referencia
  readonly formularioRef = viewChild<ElementRef<HTMLFormElement>>("formularioRef");
  
  // Signal query: required (lanza error si no existe)
  readonly campoNombre = viewChild.required<ElementRef<HTMLInputElement>>("campoNombre");
  
  // Signal query: múltiples elementos del mismo tipo
  readonly todosLosCampos = viewChildren<ElementRef<HTMLInputElement>>("campoNombre", "campoEmail");
  
  // Content query: elementos proyectados por el padre
  readonly botonesProyectados = contentChildren<ElementRef<HTMLButtonElement>>("boton");
  
  // Computed derivado de queries
  readonly cantidadCampos = computed(() => this.todosLosCampos().length);
}
```

---

## 4.6 `linkedSignal()`: Signals Derivados con Reset Automático

Angular v19 introduce `linkedSignal()`, que crea un Signal de escritura cuyo valor inicial se deriva de otro Signal, pero que puede ser sobrescrito localmente. Cuando la fuente cambia, el `linkedSignal` se **resetea** automáticamente al nuevo valor derivado:

```typescript
import { signal, linkedSignal } from "@angular/core";

interface Producto {
  id: string;
  nombre: string;
  precioBase: number;
}

const productoSeleccionado = signal<Producto>({
  id: "1", nombre: "Laptop", precioBase: 1000
});

// linkedSignal: se inicializa con el precio del producto seleccionado
// pero el usuario puede sobrescribirlo manualmente
const precioEditado = linkedSignal(() => productoSeleccionado().precioBase);

console.log(precioEditado()); // 1000

// El usuario edita el precio manualmente
precioEditado.set(950);
console.log(precioEditado()); // 950

// Cuando cambia el producto seleccionado, el precio se RESETEA
productoSeleccionado.set({ id: "2", nombre: "Monitor", precioBase: 500 });
console.log(precioEditado()); // 500 (reseteado automáticamente)
```

Caso de uso perfecto: formularios de edición donde el valor inicial viene de un Signal padre (ej. el producto seleccionado en una tabla), pero el usuario puede modificarlo localmente antes de guardar.

---

## 4.7 Resource API y `rxResource()`: Fetching Asíncrono Nativo

Angular v19.1+ introduce la **Resource API**, una forma nativa de representar datos asíncronos (peticiones HTTP, lecturas de bases de datos) como Signals reactivos, sin necesidad de suscripciones manuales ni el pipe `async`:

```typescript
import { Component, signal, inject } from "@angular/core";
import { rxResource } from "@angular/core/rxjs-interop";
import { HttpClient } from "@angular/common/http";

interface Producto {
  id: string;
  nombre: string;
  precio: number;
}

@Component({
  selector: "app-catalogo",
  standalone: true,
  template: `
    @if (productosResource.isLoading()) {
      <div class="animate-pulse">Cargando catálogo...</div>
    }
    
    @if (productosResource.error()) {
      <div class="text-red-600">
        Error: {{ productosResource.error() }}
        <button (click)="productosResource.reload()">Reintentar</button>
      </div>
    }
    
    @if (productosResource.hasValue()) {
      @for (prod of productosResource.value()!; track prod.id) {
        <div class="p-4 border rounded">
          <h3>{{ prod.nombre }}</h3>
          <p>{{ prod.precio | currency:'EUR' }}</p>
        </div>
      } @empty {
        <p>No hay productos disponibles.</p>
      }
    }
    
    <button (click)="cambiarCategoria('electronica')">Electrónica</button>
    <button (click)="cambiarCategoria('hogar')">Hogar</button>
  `
})
export class CatalogoComponent {
  private readonly http = inject(HttpClient);
  
  readonly categoriaActual = signal<string>("electronica");
  
  // rxResource: re-ejecuta la petición automáticamente cuando categoriaActual() cambia
  readonly productosResource = rxResource({
    request: () => ({ categoria: this.categoriaActual() }),
    loader: ({ request }) => {
      return this.http.get<Producto[]>(`/api/productos?cat=${request.categoria}`);
    }
  });
  
  cambiarCategoria(cat: string): void {
    this.categoriaActual.set(cat);
    // La petición HTTP se re-ejecuta automáticamente
  }
}
```

### Propiedades del Resource

| Propiedad | Tipo | Descripción |
|---|---|---|
| `value()` | `T \| undefined` | El valor resuelto del recurso |
| `isLoading()` | `boolean` | `true` mientras la petición está en vuelo |
| `error()` | `unknown` | El error si la petición falló |
| `hasValue()` | `boolean` | `true` si el recurso tiene un valor resuelto |
| `status()` | `ResourceStatus` | Estado actual: `Idle`, `Loading`, `Resolved`, `Error`, `Reloading` |
| `reload()` | Método | Fuerza una re-ejecución de la petición |

---

## 4.8 Interoperabilidad con RxJS: `toSignal()` y `toObservable()`

RxJS sigue siendo insustituible para flujos asíncronos complejos con operadores de filtrado, debounce, merge y cancelación. Angular provee puentes bidireccionales entre Signals y Observables:

### `toSignal()`: Observable → Signal

```typescript
import { Component, inject } from "@angular/core";
import { toSignal } from "@angular/core/rxjs-interop";
import { ActivatedRoute } from "@angular/router";
import { map } from "rxjs/operators";

@Component({
  selector: "app-perfil",
  standalone: true,
  template: `<h1>Perfil de: {{ userId() }}</h1>`
})
export class PerfilComponent {
  private readonly route = inject(ActivatedRoute);
  
  // Convierte el Observable de parámetros en un Signal síncrono
  readonly userId = toSignal(
    this.route.paramMap.pipe(map(params => params.get("id") ?? "desconocido")),
    { initialValue: "cargando..." }
  );
}
```

### `toObservable()`: Signal → Observable

```typescript
import { Component, signal, OnInit, inject } from "@angular/core";
import { toObservable } from "@angular/core/rxjs-interop";
import { debounceTime, distinctUntilChanged, switchMap } from "rxjs/operators";
import { HttpClient } from "@angular/common/http";

@Component({
  selector: "app-buscador-avanzado",
  standalone: true,
  template: `<input (input)="termino.set($any($event.target).value)" placeholder="Buscar...">`
})
export class BuscadorAvanzadoComponent implements OnInit {
  private readonly http = inject(HttpClient);
  readonly termino = signal("");

  ngOnInit(): void {
    toObservable(this.termino).pipe(
      debounceTime(300),
      distinctUntilChanged(),
      switchMap(t => this.http.get<string[]>(`/api/buscar?q=${t}`))
    ).subscribe(resultados => {
      console.log("Resultados:", resultados);
    });
  }
}
```

---

## 4.9 Caso de Estudio: Almacén de Estado Completo con Signals

Implementación de un servicio de estado reactivo de producción para un carrito de compras:

```typescript
import { Injectable, computed, signal, effect } from "@angular/core";

export interface ItemCarrito {
  id: string;
  nombre: string;
  precio: number;
  cantidad: number;
  imagenUrl: string;
}

@Injectable({ providedIn: "root" })
export class CarritoStateService {
  // Estado privado
  private readonly _items = signal<ItemCarrito[]>([]);
  private readonly _codigoDescuento = signal<string>("");
  
  // Estado público (solo lectura)
  readonly items = this._items.asReadonly();
  readonly codigoDescuento = this._codigoDescuento.asReadonly();
  
  // Computeds derivados
  readonly cantidadTotal = computed(() => 
    this._items().reduce((acc, item) => acc + item.cantidad, 0)
  );
  
  readonly subtotal = computed(() => 
    this._items().reduce((acc, item) => acc + item.precio * item.cantidad, 0)
  );
  
  readonly porcentajeDescuento = computed(() => {
    const codigo = this._codigoDescuento().toUpperCase();
    const descuentos: Record<string, number> = {
      "GDE_ANGULAR": 0.15,
      "WELCOME10": 0.10,
      "VIP25": 0.25
    };
    return descuentos[codigo] ?? 0;
  });
  
  readonly descuentoAplicado = computed(() => this.subtotal() * this.porcentajeDescuento());
  
  readonly totalFinal = computed(() => this.subtotal() - this.descuentoAplicado());
  
  readonly esCarritoVacio = computed(() => this.cantidadTotal() === 0);
  
  constructor() {
    // Effect: persistir en localStorage ante cada cambio
    effect(() => {
      const items = this._items();
      localStorage.setItem("carrito-items", JSON.stringify(items));
    });
    
    // Restaurar estado desde localStorage al arrancar
    const saved = localStorage.getItem("carrito-items");
    if (saved) {
      try {
        this._items.set(JSON.parse(saved));
      } catch {
        localStorage.removeItem("carrito-items");
      }
    }
  }

  agregarProducto(producto: Omit<ItemCarrito, "cantidad">): void {
    this._items.update(items => {
      const existente = items.find(i => i.id === producto.id);
      if (existente) {
        return items.map(i => 
          i.id === producto.id ? { ...i, cantidad: i.cantidad + 1 } : i
        );
      }
      return [...items, { ...producto, cantidad: 1 }];
    });
  }

  decrementarCantidad(id: string): void {
    this._items.update(items => {
      const item = items.find(i => i.id === id);
      if (item && item.cantidad <= 1) {
        return items.filter(i => i.id !== id);
      }
      return items.map(i =>
        i.id === id ? { ...i, cantidad: i.cantidad - 1 } : i
      );
    });
  }

  eliminarProducto(id: string): void {
    this._items.update(items => items.filter(i => i.id !== id));
  }

  aplicarDescuento(codigo: string): void {
    this._codigoDescuento.set(codigo);
  }

  vaciar(): void {
    this._items.set([]);
    this._codigoDescuento.set("");
  }
}
```

---

## Resumen del Capítulo

* **Angular Signals** implementa un modelo de **reactividad fina push/pull** donde los productores empujan invalidaciones y los consumidores tiran recálculos de forma perezosa, eliminando computaciones innecesarias.
* Los **Writable Signals** soportan funciones de igualdad personalizadas (`equal`) para controlar cuándo emitir notificaciones y `.asReadonly()` para proteger el estado de escrituras externas.
* Los **Computed Signals** son evaluados perezosamente, memorizados automáticamente y rastrean dependencias de forma dinámica, pudiendo cambiar el grafo de dependencias según ramas condicionales.
* Los **Effects** producen efectos secundarios reactivos, soportan limpieza de recursos con `onCleanup`, exclusión de dependencias con `untracked()` y se destruyen automáticamente con el contexto de inyección.
* Las **Signal Queries** (`viewChild`, `viewChildren`, `contentChild`, `contentChildren`) reemplazan los decoradores clásicos con APIs reactivas integradas en el grafo de Signals.
* **`linkedSignal()`** crea Signals derivados con capacidad de sobrescritura local que se resetean automáticamente cuando la fuente cambia.
* La **Resource API** (`rxResource`) proporciona una forma nativa de representar datos asíncronos como Signals, con manejo integrado de estados de carga, error y valor.
* La **interoperabilidad** con RxJS mediante `toSignal()` y `toObservable()` permite combinar la inmediatez de Signals con el poder de transformación de RxJS.

En el próximo capítulo, aprenderemos a organizar y desacoplar la lógica de negocio dominando el potente sistema de **Servicios e Inyección de Dependencias** del Angular moderno.

---

← [Capítulo anterior](03-templates-y-directivas.md) | [Inicio](README.md) | [Capítulo siguiente →](05-servicios-e-inyeccion-de-dependencias.md)
