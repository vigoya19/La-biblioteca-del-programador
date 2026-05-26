# Capítulo 8: Gestión de Estado Global a Escala

> "La arquitectura de tu estado global no define qué tan rápido corre tu aplicación hoy, sino qué tan fácil será añadirle funcionalidades mañana sin que el castillo de naipes se derrumbe."

En los capítulos anteriores aprendimos a dominar la gestión del estado a nivel de componente (`useState`) y a compartirlo mediante inyección nativa con `Context API` y `useReducer`. Sin embargo, en aplicaciones empresariales de gran escala con cientos de rutas, docenas de flujos paralelos y sincronización de datos pesados en tiempo real (por ejemplo, chats, paneles de control financiero o e-commerce), estas herramientas nativas pueden provocar problemas severos de rendimiento por re-renderizados masivos e ineficiencias de mantenimiento.

En este capítulo, analizaremos cuándo migrar a una librería de estado global, compararemos las tres grandes filosofías arquitectónicas del frontend moderno y aprenderemos a dominar **Zustand**, el estándar indiscutible de la industria actual.

---

## 8.1 Las Tres Filosofías Arquitectónicas de la Gestión de Estado

En el ecosistema de React coexisten tres formas radicalmente diferentes de modelar y manipular el estado de una aplicación a escala:

```
┌─────────────────────────────────┐ ┌─────────────────────────────────┐ ┌─────────────────────────────────┐
│ 1. ARQUITECTURA FLUX (Redux)    │ │ 2. ARQUITECTURA PROXY (Zustand) │ │ 3. ARQUITECTURA ATÓMICA (Jotai) │
│ - Flujo unidireccional rígido.  │ │ - Almacén único reactivo y      │ │ - El estado se compone de micro-│
│ - Acciones, Reducers y Store    │ │   mutable basado en selectores  │ │   nodos independientes (átomos) │
│   inmutable centralizada.       │ │   ligeros sin boilerplate.      │ │   que se asocian quirúrgicamente│
└─────────────────────────────────┘ └─────────────────────────────────┘ └─────────────────────────────────┘
```

> [!NOTE]
> ### 🏪 La Analogía del Hipermercado vs. La Tienda de Conveniencia vs. Los Post-It
> 
> Imagina que estás administrando el inventario de suministros de una gran corporación de oficinas:
> 
> - **La Arquitectura Flux (Redux)** es equivalente a un **Hipermercado corporativo centralizado (Walmart)**: Si un empleado necesita un lápiz, no puede simplemente ir a la estantería y tomarlo. Debe llenar un formulario estandarizado (**Action**), entregárselo a un camión de mensajería (**Dispatcher**), esperar a que el centro de distribución procese y verifique la factura de inventario (**Reducer**) y finalmente el almacén inmutable central (**Store**) actualiza sus registros para entregarle el lápiz. Es extremadamente seguro y blindado contra errores, pero requiere un proceso burocrático masivo (**boilerplate**).
> - **La Arquitectura de Proxies (Zustand)** es equivalente a la **Tienda de Barrio / OXXO Express**: Hay una sola ventanilla rápida atendida por un dependiente ágil. Llegas, le pides el lápiz directamente, él estira la mano y te lo entrega en un segundo (**mutación directa simplificada**). El proceso es instantáneo, liviano y no requiere formularios complejos de 4 páginas.
> - **La Arquitectura Atómica (Jotai)** es equivalente a pegar **Post-It adhesivos individuales en la pared**: No tienes un almacén central. Cada departamento pega su propia nota autoadhesiva con sus datos exactos en la pared. Si el diseñador cambia su color preferido, arranca y actualiza su Post-It exclusivo (**átomo**). Ninguno de los programadores, administradores u otros departamentos se enteran o tienen que detener sus labores, logrando una reactividad quirúrgica y aislada en cada componente.

---

## 8.2 Zustand: La Simplificación Moderna

**Zustand** (que significa *"estado"* en alemán) es una librería de estado ultra-liviana (~1.5KB gzipped), basada en una filosofía de store centralizada pero extremadamente simplificada mediante el uso de ganchos (*hooks*) de JavaScript y proxies en memoria.

*   **Sin Boilerplate**: No necesitas reducers, action creators, dispatchers ni envolver tu aplicación completa en proveedores de contexto (`<Provider>`).
*   **Actualizaciones Quirúrgicas**: Zustand utiliza **selectores** de estado. Un componente solo se re-renderizará si la propiedad específica del selector cambia.
*   **Fuera del flujo de React**: El estado de Zustand reside fuera del Virtual DOM de React, lo que le permite integrarse de forma nativa con JavaScript convencional y APIs de backend asíncronas de manera directa.

---

## 8.3 Creación de una Store de Carrito de Compras en TypeScript

Implementemos un almacén dinámico para un carrito de compras profesional con Zustand:

### `cartStore.ts`
```typescript
import { create } from 'zustand';
import { persist } from 'zustand/middleware';

// 1. Declarar la interfaz estricta del Producto y el Estado de la Store
export interface Product {
  id: string;
  name: string;
  price: number;
}

export interface CartItem extends Product {
  quantity: number;
}

interface CartState {
  items: CartItem[];
  addItem: (product: Product) => void;
  removeItem: (id: string) => void;
  clearCart: () => void;
  getTotalItems: () => number;
  getTotalPrice: () => number;
}

// 2. Crear la Store con Zustand y agregar middleware de persistencia en LocalStorage
export const useCartStore = create<CartState>()(
  persist(
    (set, get) => ({
      items: [],
      
      // Añadir o incrementar producto en el carrito
      addItem: (product: Product) => {
        const currentItems = get().items;
        const existingItem = currentItems.find(item => item.id === product.id);

        if (existingItem) {
          set({
            items: currentItems.map(item =>
              item.id === product.id ? { ...item, quantity: item.quantity + 1 } : item
            )
          });
        } else {
          set({ items: [...currentItems, { ...product, quantity: 1 }] });
        }
      },

      // Remover o decrementar un producto del carrito
      removeItem: (id: string) => {
        const currentItems = get().items;
        const existingItem = currentItems.find(item => item.id === id);

        if (existingItem && existingItem.quantity > 1) {
          set({
            items: currentItems.map(item =>
              item.id === id ? { ...item, quantity: item.quantity - 1 } : item
            )
          });
        } else {
          set({ items: currentItems.filter(item => item.id !== id) });
        }
      },

      // Vaciar carrito
      clearCart: () => set({ items: [] }),

      // Métodos derivados computados reactivamente
      getTotalItems: () => {
        return get().items.reduce((total, item) => total + item.quantity, 0);
      },

      getTotalPrice: () => {
        return get().items.reduce((total, item) => total + item.price * item.quantity, 0);
      }
    }),
    {
      name: 'cart-storage' // Nombre de la clave física en LocalStorage para auto-guardado
    }
  )
);
```

---

## 8.4 Consumo Eficiente mediante Selectores

Para evitar que tu componente se re-renderice innecesariamente cuando cambien propiedades de la store que no consume, **debes usar selectores específicos de forma obligatoria**:

```typescript
import React from 'react';
import { useCartStore } from './store/cartStore';

export function ComponenteNavbar(): React.JSX.Element {
  // ❌ INCORRECTO: Re-renderiza el Navbar completo por CUALQUIER cambio en la store
  // const store = useCartStore(); 

  //  CORRECTO: El Navbar solo se volverá a pintar si cambia el conteo total de items
  const totalItems = useCartStore((state) => state.getTotalItems());
  const clearCart = useCartStore((state) => state.clearCart);

  return (
    <nav style={{ padding: '1rem', borderBottom: '1px solid #ccc', display: 'flex', justifyContent: 'space-between' }}>
      <h3>Mi Tienda Online</h3>
      <div>
        <span>🛒 Carrito: {totalItems} items</span>
        <button onClick={clearCart} style={{ marginLeft: '1rem' }}>Vaciar</button>
      </div>
    </nav>
  );
}
```

---

## Resumen del Capítulo

*   Las aplicaciones empresariales complejas requieren stores de estado global para desacoplar el estado del ciclo de vida y renders de las páginas individuales.
*   **Zustand** lidera el mercado moderno frontend por su **ausencia de boilerplate**, peso ultra-ligero y reactividad basada en selectores quirúrgicos.
*   El uso de **selectores en Zustand** es el mecanismo fundamental para garantizar que los componentes se actualicen de forma óptima sin arrastrar renderizados innecesarios en cascada.
*   Los middlewares integrados como **`persist`** nos permiten automatizar la sincronización de estado directamente con LocalStorage, cookies o SessionStorage de forma transparente para el desarrollador.

En el próximo capítulo, ingresaremos al fascinante universo del renderizado no bloqueante, analizando cómo React v18 y v19 orquestan la prioridad de los renders concurrentes utilizando **Suspense**, **useTransition** y el nuevo hook de resolución de promesas **`use`**.

---

[Capítulo anterior](07-usereducer.md) | [Inicio](README.md) | [Capítulo siguiente →](09-suspense-y-concurrencia.md)
