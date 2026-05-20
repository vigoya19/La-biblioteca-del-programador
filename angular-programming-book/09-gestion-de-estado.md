# Capítulo 9: Gestión de Estado

> "El estado es la fuente de todos los bugs y de todas las features. Gestionarlo correctamente es la habilidad más valiosa de un arquitecto frontend."

A medida que una aplicación web escala en tamaño y complejidad, la gestión del flujo de información y la sincronización de datos entre múltiples componentes no relacionados se convierte en uno de los mayores desafíos del desarrollo frontend. Sin una arquitectura clara, las aplicaciones tienden a sufrir de problemas como el "Prop Drilling" (pasar propiedades a través de múltiples niveles de componentes intermedios), inconsistencias graves de datos en pantalla, o el temido "estado fantasma" donde la UI muestra datos obsoletos que ya no reflejan la realidad del servidor.

En este capítulo, aprenderemos a diseñar la arquitectura de estado correcta para cada escenario, desde el más simple hasta el más complejo. Exploraremos tres niveles de complejidad de forma exhaustiva:

1. **State Services con Signals**: Para equipos pequeños y aplicaciones de complejidad baja-media.
2. **NgRx Signals Store**: Para aplicaciones empresariales con equipos medianos y múltiples dominios.
3. **NgRx Store clásico (Redux)**: Para organizaciones masivas con requisitos estrictos de trazabilidad.

Además, compararemos alternativas emergentes del ecosistema (**Elf**, **StateAdapt**) para que puedas tomar decisiones arquitectónicas informadas.

---

## 9.1 ¿Cuándo usar Estado Local, Servicios con Signals o un Store Global?

No todas las partes de tu aplicación requieren la complejidad de un almacén global. Como arquitecto de software, debes aplicar el principio de **simplicidad pragmática**:

```
            ┌───────────────────────────────────────────────┐
            │          ¿Qué alcance tiene el dato?          │
            └───────────────────────┬───────────────────────┘
                                    │
         ┌──────────────────────────┼──────────────────────────┐
         ▼                          ▼                          ▼
 ┌───────────────┐          ┌───────────────┐          ┌───────────────┐
 │ Local / Vista │          │ Compartido    │          │ Global        │
 │ (Una pantalla)│          │ (Módulo/Ruta) │          │ (Toda la App) │
 └───────┬───────┘          └───────┬───────┘          └───────┬───────┘
         ▼                          ▼                          ▼
 ┌───────────────┐          ┌───────────────┐          ┌───────────────┐
 │ Signal local  │          │ Servicio DI   │          │ NgRx Signals  │
 │ `signal()`    │          │ con Signals   │          │     Store     │
 └───────────────┘          └───────────────┘          └───────────────┘
```

| Nivel | Ejemplo | Solución | Complejidad |
|---|---|---|---|
| **Local** | ¿Modal abierto? Término de búsqueda temporal. Validación de un input. | `signal()` dentro del componente | Ninguna |
| **Compartido** | Wizard de 3 pasos. Datos de un formulario multi-sección. | Servicio `@Injectable` con Signals, provisto a nivel de componente o ruta | Baja |
| **Global** | Sesión del usuario, carrito de compras, configuración de idioma, notificaciones. | `signalStore` con `providedIn: 'root'` | Media |
| **Global + Trazabilidad** | Estado bancario con auditoría de cada cambio, requisitos legales de logging de acciones. | NgRx Store clásico con Redux DevTools | Alta |

> **Regla de oro**: Empieza siempre con el nivel más simple posible. Si un `signal()` local resuelve tu problema, no lo metas en un Store global. La sobre-ingeniería es tan perjudicial como la sub-ingeniería.

---

## 9.2 Patrón de State Service con Signals (Sin Dependencias Externas)

Antes de adoptar cualquier librería de terceros, debes dominar el patrón más sencillo y poderoso que Angular ofrece nativamente: un servicio `@Injectable` que encapsula estado con Signals y expone derivaciones con `computed()`.

### Implementación Completa: TodoService

```typescript
import { Injectable, signal, computed, effect } from "@angular/core";

export interface Todo {
  id: string;
  titulo: string;
  completado: boolean;
  prioridad: "alta" | "media" | "baja";
  creadoEn: Date;
}

type FiltroTodo = "todos" | "activos" | "completados";

@Injectable({ providedIn: "root" })
export class TodoStateService {
  // ── Estado privado (solo este servicio puede modificarlo) ──
  private readonly _todos = signal<Todo[]>([]);
  private readonly _filtroActual = signal<FiltroTodo>("todos");
  private readonly _terminoBusqueda = signal("");
  
  // ── Estado público de solo lectura ──
  readonly todos = this._todos.asReadonly();
  readonly filtroActual = this._filtroActual.asReadonly();
  readonly terminoBusqueda = this._terminoBusqueda.asReadonly();
  
  // ── Derivaciones computadas (se re-calculan automáticamente) ──
  readonly todosFiltrados = computed(() => {
    let resultado = this._todos();
    
    // Filtrar por estado
    const filtro = this._filtroActual();
    if (filtro === "activos") resultado = resultado.filter(t => !t.completado);
    if (filtro === "completados") resultado = resultado.filter(t => t.completado);
    
    // Filtrar por búsqueda
    const termino = this._terminoBusqueda().toLowerCase();
    if (termino) resultado = resultado.filter(t => t.titulo.toLowerCase().includes(termino));
    
    return resultado;
  });
  
  readonly estadisticas = computed(() => {
    const todos = this._todos();
    return {
      total: todos.length,
      completados: todos.filter(t => t.completado).length,
      activos: todos.filter(t => !t.completado).length,
      porcentaje: todos.length ? Math.round((todos.filter(t => t.completado).length / todos.length) * 100) : 0,
      porPrioridad: {
        alta: todos.filter(t => t.prioridad === "alta" && !t.completado).length,
        media: todos.filter(t => t.prioridad === "media" && !t.completado).length,
        baja: todos.filter(t => t.prioridad === "baja" && !t.completado).length
      }
    };
  });

  constructor() {
    // Effect: persistir automáticamente en localStorage
    effect(() => {
      const datos = this._todos();
      localStorage.setItem("app-todos", JSON.stringify(datos));
    });
    
    // Restaurar al arrancar
    const guardados = localStorage.getItem("app-todos");
    if (guardados) {
      try { this._todos.set(JSON.parse(guardados)); } catch { /* silencioso */ }
    }
  }

  // ── Acciones (métodos públicos que modifican el estado) ──
  agregar(titulo: string, prioridad: "alta" | "media" | "baja" = "media"): void {
    const nuevo: Todo = {
      id: crypto.randomUUID(),
      titulo: titulo.trim(),
      completado: false,
      prioridad,
      creadoEn: new Date()
    };
    this._todos.update(lista => [nuevo, ...lista]);
  }

  toggleCompletado(id: string): void {
    this._todos.update(lista =>
      lista.map(t => t.id === id ? { ...t, completado: !t.completado } : t)
    );
  }

  eliminar(id: string): void {
    this._todos.update(lista => lista.filter(t => t.id !== id));
  }

  setFiltro(filtro: FiltroTodo): void {
    this._filtroActual.set(filtro);
  }

  setBusqueda(termino: string): void {
    this._terminoBusqueda.set(termino);
  }

  limpiarCompletados(): void {
    this._todos.update(lista => lista.filter(t => !t.completado));
  }
}
```

### Ventajas del Patrón State Service

| Ventaja | Detalle |
|---|---|
| **Cero dependencias** | No requiere instalar ninguna librería |
| **Tipado nativo** | TypeScript completo sin tipos especiales |
| **Familiar** | Usa patrones estándar de Angular (servicios, DI) |
| **Testeable** | Se prueba como cualquier servicio con `TestBed` |
| **Rendimiento** | Signals + `computed()` ofrecen reactividad granular óptima |

### Limitaciones del Patrón State Service

- **Sin DevTools**: No hay forma visual de inspeccionar el historial de cambios de estado.
- **Sin trazabilidad**: No hay concepto de "acciones" discretas para auditar qué cambió y por qué.
- **Sin undo/redo**: No hay historial de estados previos.
- **Escalabilidad**: Con 20+ signals en un solo servicio, el código se vuelve difícil de mantener.

---

## 9.3 NgRx Signals Store (`@ngrx/signals`): La Solución Moderna

Para resolver las limitaciones del State Service mientras se mantiene la simplicidad, el equipo de NgRx lanzó **NgRx Signals Store**: una API funcional y modular basada en composición de features.

```bash
npm install @ngrx/signals
```

### Creación de un Store Completo

```typescript
import { inject, computed } from "@angular/core";
import { signalStore, withState, withComputed, withMethods, withHooks, patchState } from "@ngrx/signals";
import { rxMethod } from "@ngrx/signals/rxjs-interop";
import { pipe, switchMap, tap, catchError, of } from "rxjs";
import { HttpClient } from "@angular/common/http";

export interface Usuario {
  id: string;
  nombre: string;
  email: string;
  rol: "admin" | "editor" | "viewer";
  activo: boolean;
}

export interface UsuarioState {
  usuarios: Usuario[];
  cargando: boolean;
  error: string | null;
  filtroRol: string;
  terminoBusqueda: string;
}

const estadoInicial: UsuarioState = {
  usuarios: [],
  cargando: false,
  error: null,
  filtroRol: "todos",
  terminoBusqueda: ""
};

export const UsuarioStore = signalStore(
  { providedIn: "root" },

  withState(estadoInicial),

  withComputed(({ usuarios, filtroRol, terminoBusqueda }) => ({
    usuariosFiltrados: computed(() => {
      let resultado = usuarios();
      const rol = filtroRol();
      if (rol !== "todos") resultado = resultado.filter(u => u.rol === rol);
      const termino = terminoBusqueda().toLowerCase();
      if (termino) resultado = resultado.filter(u =>
        u.nombre.toLowerCase().includes(termino) || u.email.toLowerCase().includes(termino)
      );
      return resultado;
    }),
    totalUsuarios: computed(() => usuarios().length),
    totalActivos: computed(() => usuarios().filter(u => u.activo).length),
    totalPorRol: computed(() => ({
      admin: usuarios().filter(u => u.rol === "admin").length,
      editor: usuarios().filter(u => u.rol === "editor").length,
      viewer: usuarios().filter(u => u.rol === "viewer").length
    }))
  })),

  withMethods((store) => {
    const http = inject(HttpClient);

    return {
      // Métodos síncronos
      setFiltroRol(rol: string): void {
        patchState(store, { filtroRol: rol });
      },
      
      setBusqueda(termino: string): void {
        patchState(store, { terminoBusqueda: termino });
      },

      // Método asíncrono con rxMethod
      cargarUsuarios: rxMethod<void>(
        pipe(
          tap(() => patchState(store, { cargando: true, error: null })),
          switchMap(() => http.get<Usuario[]>("/api/usuarios").pipe(
            tap(usuarios => patchState(store, { usuarios, cargando: false })),
            catchError(err => {
              patchState(store, { cargando: false, error: err.message });
              return of(null);
            })
          ))
        )
      ),
      
      // CRUD local optimista (actualiza UI primero, luego sincroniza con el servidor)
      toggleActivo(userId: string): void {
        // 1. Actualización optimista inmediata
        patchState(store, (state) => ({
          usuarios: state.usuarios.map(u =>
            u.id === userId ? { ...u, activo: !u.activo } : u
          )
        }));
        
        // 2. Sincronizar con el servidor (en background)
        const usuario = store.usuarios().find(u => u.id === userId);
        if (usuario) {
          http.patch(`/api/usuarios/${userId}`, { activo: usuario.activo }).subscribe({
            error: () => {
              // 3. Revertir si falla
              patchState(store, (state) => ({
                usuarios: state.usuarios.map(u =>
                  u.id === userId ? { ...u, activo: !u.activo } : u
                ),
                error: "No se pudo actualizar el estado del usuario"
              }));
            }
          });
        }
      }
    };
  }),

  withHooks({
    onInit(store) {
      store.cargarUsuarios();
    }
  })
);
```

---

## 9.4 Custom Store Features Reutilizables

El poder real de NgRx Signals Store reside en la capacidad de crear **features reutilizables** que encapsulan patrones comunes (carga, paginación, persistencia) y se pueden componer en cualquier Store.

### Feature: `withLoading()`

```typescript
import { computed } from "@angular/core";
import { signalStoreFeature, withState, withComputed, withMethods, patchState } from "@ngrx/signals";

export interface LoadingState {
  cargando: boolean;
  error: string | null;
}

export function withLoading() {
  return signalStoreFeature(
    withState<LoadingState>({ cargando: false, error: null }),
    withComputed(({ cargando, error }) => ({
      tieneError: computed(() => error() !== null)
    })),
    withMethods((store) => ({
      iniciarCarga(): void {
        patchState(store, { cargando: true, error: null });
      },
      finalizarCarga(): void {
        patchState(store, { cargando: false });
      },
      reportarError(mensaje: string): void {
        patchState(store, { cargando: false, error: mensaje });
      }
    }))
  );
}
```

### Feature: `withPagination()`

```typescript
import { computed } from "@angular/core";
import { signalStoreFeature, withState, withComputed, withMethods, patchState } from "@ngrx/signals";

export interface PaginationState {
  paginaActual: number;
  elementosPorPagina: number;
  totalElementos: number;
}

export function withPagination(elementosPorPagina = 10) {
  return signalStoreFeature(
    withState<PaginationState>({
      paginaActual: 1,
      elementosPorPagina,
      totalElementos: 0
    }),
    withComputed(({ paginaActual, elementosPorPagina, totalElementos }) => ({
      totalPaginas: computed(() => Math.ceil(totalElementos() / elementosPorPagina())),
      offset: computed(() => (paginaActual() - 1) * elementosPorPagina()),
      esPrimeraPagina: computed(() => paginaActual() === 1),
      esUltimaPagina: computed(() => {
        const total = Math.ceil(totalElementos() / elementosPorPagina());
        return paginaActual() >= total;
      })
    })),
    withMethods((store) => ({
      irAPagina(pagina: number): void {
        patchState(store, { paginaActual: pagina });
      },
      siguientePagina(): void {
        patchState(store, (state) => ({ paginaActual: state.paginaActual + 1 }));
      },
      paginaAnterior(): void {
        patchState(store, (state) => ({
          paginaActual: Math.max(1, state.paginaActual - 1)
        }));
      },
      setTotalElementos(total: number): void {
        patchState(store, { totalElementos: total });
      }
    }))
  );
}
```

### Feature: `withLocalStorage()`

```typescript
import { effect } from "@angular/core";
import { signalStoreFeature, withHooks, getState } from "@ngrx/signals";

export function withLocalStorage<T extends object>(key: string) {
  return signalStoreFeature(
    withHooks({
      onInit(store: any) {
        // Restaurar estado desde localStorage
        const saved = localStorage.getItem(key);
        if (saved) {
          try {
            const parsed = JSON.parse(saved);
            Object.keys(parsed).forEach(k => {
              if (typeof store[k] === "function" && "set" in store[k]) {
                // Intentar restaurar cada señal
              }
            });
          } catch { /* ignorar datos corruptos */ }
        }
        
        // Persistir automáticamente ante cambios
        effect(() => {
          const state = getState(store);
          localStorage.setItem(key, JSON.stringify(state));
        });
      }
    })
  );
}
```

### Composición de Features en un Store

```typescript
export const ProductoStore = signalStore(
  { providedIn: "root" },
  
  withState({ productos: [] as Producto[] }),
  withLoading(),           // ← Feature reutilizable de loading
  withPagination(20),      // ← Feature reutilizable de paginación (20 items/página)
  
  withComputed(({ productos, paginaActual, elementosPorPagina }) => ({
    productosPaginados: computed(() => {
      const offset = (paginaActual() - 1) * elementosPorPagina();
      return productos().slice(offset, offset + elementosPorPagina());
    })
  })),
  
  withMethods((store) => ({
    // Los métodos de withLoading() y withPagination() ya están disponibles aquí
    // store.iniciarCarga(), store.finalizarCarga(), store.siguientePagina(), etc.
  }))
);
```

---

## 9.5 Entities Management con `@ngrx/signals/entities`

Para stores que gestionan colecciones de entidades (usuarios, productos, pedidos), NgRx provee `withEntities()`, que genera automáticamente métodos CRUD optimizados:

```typescript
import { signalStore, withMethods } from "@ngrx/signals";
import { withEntities, addEntity, updateEntity, removeEntity, setAllEntities } from "@ngrx/signals/entities";

export interface Notificacion {
  id: string;
  titulo: string;
  mensaje: string;
  leida: boolean;
  timestamp: Date;
}

export const NotificacionStore = signalStore(
  { providedIn: "root" },
  
  // withEntities genera automáticamente: entities(), ids(), entityMap()
  withEntities<Notificacion>(),
  
  withMethods((store) => ({
    agregar(notif: Notificacion): void {
      // addEntity añade la entidad al mapa interno de forma optimizada
      addEntity(store, notif);
    },
    
    marcarComoLeida(id: string): void {
      updateEntity(store, { id, changes: { leida: true } });
    },
    
    eliminar(id: string): void {
      removeEntity(store, id);
    },
    
    cargarTodas(notificaciones: Notificacion[]): void {
      setAllEntities(store, notificaciones);
    }
  }))
);
```

---

## 9.6 Comparativa de Soluciones: ¿Cuándo usar Cada Una?

### Tabla Comparativa Detallada

| Criterio | State Service + Signals | NgRx Signals Store | NgRx Store (Redux) | Elf |
|---|---|---|---|---|
| **Dependencias** | Ninguna | `@ngrx/signals` | `@ngrx/store`, `@ngrx/effects`, `@ngrx/entity` | `@ngneat/elf` |
| **Boilerplate** | Mínimo | Bajo | Alto (Actions, Reducers, Effects) | Bajo-Medio |
| **Curva de aprendizaje** | Ninguna (Signals nativos) | Baja (API funcional) | Alta (patrón Redux completo) | Baja |
| **DevTools** | ❌ No | ✅ Parcial (`@ngrx/store-devtools`) | ✅ Completo (Redux DevTools) | ✅ Propio |
| **Trazabilidad de acciones** | ❌ No | ❌ No (métodos directos) | ✅ Sí (acciones discretas con tipo) | ❌ No |
| **Undo/Redo** | ❌ Manual | ❌ Manual | ✅ Con meta-reducer | ❌ Manual |
| **Server-side compatible** | ✅ | ✅ | ✅ | ✅ |
| **Tamaño del equipo ideal** | 1-5 devs | 3-15 devs | 10+ devs | 2-10 devs |
| **Reactividad** | Signals (fine-grained) | Signals (fine-grained) | RxJS (Observables) | RxJS (Observables) |

### Guía de Decisión

```
¿Tu app tiene más de 3 dominios de estado global?
│
├── NO → State Service con Signals ✅
│         (Suficiente para la mayoría de apps)
│
└── SÍ → ¿Necesitas DevTools y trazabilidad de acciones?
          │
          ├── NO → NgRx Signals Store ✅
          │         (La mejor relación poder/simplicidad)
          │
          └── SÍ → ¿Necesitas auditoría legal de cada cambio?
                    │
                    ├── NO → NgRx Signals Store + custom logging ✅
                    │
                    └── SÍ → NgRx Store clásico (Redux) ✅
                              (Máxima trazabilidad y herramientas)
```

### Sobre Elf (de ngneat)

**Elf** es una alternativa ligera y elegante de gestión de estado desarrollada por el equipo de ngneat. Se basa en RxJS y ofrece una API fluida con stores tipados:

```typescript
// Ejemplo con Elf (para referencia)
import { createStore, withProps, select } from "@ngneat/elf";
import { withEntities, selectAllEntities, addEntities } from "@ngneat/elf-entities";

interface ProductoProps { filtro: string; }

const productoStore = createStore(
  { name: "productos" },
  withProps<ProductoProps>({ filtro: "" }),
  withEntities<Producto>()
);

// Consulta reactiva
const productos$ = productoStore.pipe(selectAllEntities());
```

**Ventajas de Elf**: API más concisa que NgRx clásico, stores independientes sin configuración global, soporte de persistencia integrado.

**Desventaja**: Basado completamente en RxJS (sin soporte nativo de Signals), menor comunidad y documentación que NgRx.

> **Recomendación del autor**: Para proyectos Angular modernos (v17+), **NgRx Signals Store** es la solución ideal en el 90% de los casos. Ofrece la mejor integración con el ecosistema de Signals, composición de features reutilizables, y tiene el respaldo del equipo que más entiende la arquitectura interna de Angular.

---

## 9.7 DevTools y Depuración de Estado

### Redux DevTools con NgRx

Si usas NgRx Store clásico, puedes instalar `@ngrx/store-devtools` para inspeccionar el estado completo, viajar en el tiempo entre acciones y depurar cambios:

```typescript
// app.config.ts
import { provideStoreDevtools } from "@ngrx/store-devtools";
import { isDevMode } from "@angular/core";

export const appConfig: ApplicationConfig = {
  providers: [
    // ...
    provideStoreDevtools({
      maxAge: 50, // Retiene últimas 50 acciones
      logOnly: !isDevMode(), // Solo logging en producción
      autoPause: true // Pausar al perder el foco del navegador
    })
  ]
};
```

### Depuración Manual de Signals Stores

Para NgRx Signals Store y State Services, puedes crear un utility de depuración:

```typescript
import { effect, isDevMode } from "@angular/core";
import { getState } from "@ngrx/signals";

export function debugStore(store: any, nombre: string): void {
  if (isDevMode()) {
    effect(() => {
      const state = getState(store);
      console.groupCollapsed(`[${nombre}] Estado actualizado`);
      console.log(JSON.parse(JSON.stringify(state)));
      console.groupEnd();
    });
  }
}

// Uso en el onInit del store:
withHooks({
  onInit(store) {
    debugStore(store, "UsuarioStore");
  }
})
```

---

## Resumen del Capítulo

* La **Gestión de Estado** debe estructurarse pragmáticamente según el alcance del dato: **Signals locales** para vistas aisladas, **State Services** para módulos compartidos, **NgRx Signals Store** para estado global empresarial.
* El patrón de **State Service con Signals** es la solución más simple y nativa: cero dependencias, tipado completo, persistencia con `effect()`. Suficiente para el 70% de las aplicaciones.
* **NgRx Signals Store** representa la evolución del ecosistema NgRx: API funcional basada en composición de features (`withState`, `withComputed`, `withMethods`, `withHooks`), eliminando el boilerplate de Redux.
* Las **Custom Store Features** (`withLoading()`, `withPagination()`, `withLocalStorage()`) encapsulan patrones comunes y se componen en cualquier Store, promoviendo la reutilización.
* **`withEntities()`** de `@ngrx/signals/entities` genera métodos CRUD optimizados para colecciones de entidades.
* **NgRx Store clásico** sigue siendo la opción correcta para organizaciones masivas con requisitos estrictos de trazabilidad, auditoría y time-travel debugging.
* Alternativas como **Elf** ofrecen APIs elegantes basadas en RxJS pero carecen de integración nativa con Signals.
* Las **DevTools** son esenciales para depurar estado en aplicaciones complejas: Redux DevTools para NgRx clásico, utilities personalizados para Signals Stores.

En el próximo capítulo, profundizaremos en el comportamiento interno de Angular analizando su **Ciclo de Vida en Detalle** y dominando el desarrollo de alto rendimiento mediante **Zoneless Angular** y la eliminación de `zone.js`.

---

← [Capítulo anterior](08-comunicacion-asincrona-http.md) | [Inicio](README.md) | [Capítulo siguiente →](10-ciclo-de-vida-y-optimizaciones.md)
