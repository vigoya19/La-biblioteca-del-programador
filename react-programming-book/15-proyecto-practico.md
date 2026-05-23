# Capítulo 15: Proyecto Práctico Completo e Integrador

> "El verdadero valor de la teoría de la ingeniería de software solo se consolida cuando se aplica para estructurar un sistema complejo del mundo real listo para ser escalado por múltiples equipos en producción."

A lo largo de este libro, hemos estudiado en profundidad las bases de la UI declarativa, el motor del Virtual DOM, el ciclo de vida de los componentes, la inyección mediante Context, la orquestación del estado (local con `useState`/`useReducer` y global con **Zustand**), las características concurrentes de React 18/19, la arquitectura híbrida fullstack (RSC) y las validaciones de negocio estrictas con **React Hook Form + Zod**.

En este capítulo final integrador, uniremos cada una de estas piezas y patrones de diseño para construir una **Aplicación E-Commerce Premium de Alto Rendimiento**, estructurada bajo una **Arquitectura Feature-Driven** profesional.

---

## 15.1 La Arquitectura Feature-Driven

En proyectos corporativos grandes, organizar los archivos de la aplicación por "tipo de archivo" (poner todos los componentes en `/components`, todos los hooks en `/hooks` y todos los estilos en `/styles`) se vuelve insostenible. Múltiples equipos modificando la misma carpeta provocan conflictos de git continuos y dificultan el aislamiento de código.

La **Arquitectura Feature-Driven** organiza la aplicación por **funcionalidades de negocio**. Cada funcionalidad es un módulo autocontenido que agrupa sus propios componentes de presentación, hooks locales, stores de estado, estilos y pruebas de integración.

```
src/
├── features/
│   ├── catalog/              # Módulo de Catálogo de Productos
│   │   ├── components/       # Componentes de presentación (Filtros, Cards)
│   │   ├── hooks/            # Custom Hooks de fetching del catálogo
│   │   └── index.ts          # API pública expuesta del catálogo
│   ├── cart/                 # Módulo del Carrito de Compras
│   │   ├── components/       # Modal del carrito, Navbar badge
│   │   └── cartStore.ts      # Store de Zustand del carrito
│   └── checkout/             # Módulo de Formulario de Pago y Envío
│       ├── components/       # Formulario con Hook Form y Zod
│       └── checkoutSchema.ts # Esquema estricto de validación
├── router/                   # Configuración de React Router
└── App.tsx                   # Componente raíz
```

> [!NOTE]
> ### 🏙️ La Analogía del Rascacielos Modular Completo
> 
> Imagine que eres el Urbanista en Jefe encargado de diseñar una metrópolis moderna de rascacielos inteligentes:
> 
> - Una estructura desorganizada tradicional (**Monolito de carpetas**) es equivalente a mezclar todos los ladrillos, tuberías de desagüe, cables eléctricos de alta tensión y sofás de la ciudad completa en un **único almacén central gigantesco**. Si necesitas cambiar un interruptor de luz en un rascacielos residencial, debes ir al almacén a rebuscar y corres el riesgo de cortar la luz del hospital de la ciudad por accidente.
> - **Feature-Driven Architecture** es equivalente a construir **Rascacielos Modulares Autocontenidos (Features)**:
> - El rascacielos residencial tiene su propia subestación eléctrica, sus ascensores privados y su plano de tuberías interno (**catalog, cart, checkout**).
> - Si decides remodelar la recepción del edificio de oficinas, el hospital de al lado sigue operando al 100% de forma segura y sin riesgo de caídas de servicio. Logras una ciudad modular, limpia, segura de escalar por múltiples cuadrillas de obreros de forma autónoma.

---

## 15.2 Implementación del Core del E-Commerce

A continuación, delineamos la integración de las tecnologías clave de nuestro proyecto interactivo:

### 1. El Catálogo con Búsqueda Concurrente no Bloqueante
En el módulo `catalog`, implementamos la visualización y filtrado de productos. Empleamos el hook **`useTransition`** (estudiado en el Capítulo 9) para asegurar que el input de búsqueda del catálogo sea 100% fluido y responsivo, sin importar si filtramos miles de artículos pesados en memoria:

```typescript
// catalog/components/ListadoProductos.tsx
import React, { useState, useTransition } from 'react';
import { useCartStore } from '../../features/cart/cartStore';

const PRODUCTOS_DATABASE = [
  { id: 'p1', name: 'Laptop Pro de Desarrollo', price: 1200 },
  { id: 'p2', name: 'Teclado Mecánico RGB Cherry', price: 150 },
  { id: 'p3', name: 'Monitor Curvo 4K IPS', price: 450 },
  { id: 'p4', name: 'Ratón Ergonómico Inalámbrico', price: 80 }
];

export function ListadoProductos(): React.JSX.Element {
  const [filtro, setFiltro] = useState<string>('');
  const [productosFiltrados, setProductosFiltrados] = useState(PRODUCTOS_DATABASE);
  const [isPending, startTransition] = useTransition();

  // Consumir la acción de añadir al carrito desde Zustand
  const addItem = useCartStore((state) => state.addItem);

  const handleBuscar = (e: React.ChangeEvent<HTMLInputElement>) => {
    const valor = e.target.value;
    setFiltro(valor);

    // Búsqueda concurrente no bloqueante de baja prioridad
    startTransition(() => {
      const filtrados = PRODUCTOS_DATABASE.filter(p =>
        p.name.toLowerCase().includes(valor.toLowerCase())
      );
      setProductosFiltrados(filtrados);
    });
  };

  return (
    <section style={{ padding: '1rem' }}>
      <input 
        type="text" 
        value={filtro} 
        onChange={handleBuscar} 
        placeholder="Buscar productos en catálogo..."
        style={{ width: '100%', padding: '0.75rem', fontSize: '1rem', marginBottom: '1.5rem' }}
      />

      {isPending && <p>Actualizando catálogo en segundo plano...</p>}

      <div style={{ display: 'grid', gridTemplateColumns: 'repeat(auto-fill, minmax(200px, 1fr))', gap: '1rem' }}>
        {productosFiltrados.map(prod => (
          <div key={prod.id} style={{ border: '1px solid #ddd', padding: '1rem', borderRadius: '8px' }}>
            <h4>{prod.name}</h4>
            <p style={{ fontWeight: 'bold' }}>${prod.price}</p>
            <button 
              onClick={() => addItem(prod)}
              style={{ padding: '0.5rem', cursor: 'pointer', background: '#0070f3', color: '#fff', border: 'none', borderRadius: '4px' }}
            >
              Añadir al Carrito
            </button>
          </div>
        ))}
      </div>
    </section>
  );
}
```

### 2. El Checkout Validado con React Hook Form + Zod
En el módulo `checkout`, el usuario completa su información de pago. Usamos **React Hook Form + Zod** (estudiados en el Capítulo 12) para validar que el código de seguridad de la tarjeta y la dirección de envío sigan esquemas corporativos estrictos antes de simular la pasarela de pago:

```typescript
// checkout/components/FormularioCheckout.tsx
import React from 'react';
import { useForm } from 'react-hook-form';
import { zodResolver } from '@hookform/resolvers/zod';
import { z } from 'zod';
import { useCartStore } from '../../features/cart/cartStore';

const esquemaCheckout = z.object({
  direccion: z.string().min(5, 'Especifica una dirección de envío física detallada'),
  tarjeta: z.string().regex(/^[0-9]{16}$/, 'El número de tarjeta debe tener 16 dígitos numéricos'),
  cvv: z.string().regex(/^[0-9]{3}$/, 'El CVV debe tener 3 dígitos')
});

type CheckoutData = z.infer<typeof esquemaCheckout>;

export function FormularioCheckout(): React.JSX.Element {
  const { register, handleSubmit, formState: { errors, isSubmitting } } = useForm<CheckoutData>({
    resolver: zodResolver(esquemaCheckout)
  });
  
  const clearCart = useCartStore((state) => state.clearCart);
  const items = useCartStore((state) => state.items);

  const alProcesarPago = async (data: CheckoutData) => {
    if (items.length === 0) {
      alert('Tu carrito está vacío.');
      return;
    }
    await new Promise(resolve => setTimeout(resolve, 2000));
    console.log('Pago procesado con Zod y Zustand:', data);
    alert('¡Transacción completada! Muchas gracias por tu compra.');
    clearCart(); // Vaciar carrito desde Zustand
  };

  return (
    <form onSubmit={handleSubmit(alProcesarPago)} style={{ maxWidth: '400px', margin: '2rem auto', display: 'flex', flexDirection: 'column', gap: '1rem' }}>
      <h3>Formulario de Compra Segura</h3>
      
      <div>
        <label htmlFor="direccion" style={{ display: 'block', fontWeight: 'bold' }}>Dirección de Envío:</label>
        <input id="direccion" type="text" {...register('direccion')} style={{ width: '100%', padding: '0.5rem' }} />
        {errors.direccion && <span style={{ color: 'red', fontSize: '0.85rem' }}>{errors.direccion.message}</span>}
      </div>

      <div>
        <label htmlFor="tarjeta" style={{ display: 'block', fontWeight: 'bold' }}>Número de Tarjeta (16 dígitos):</label>
        <input id="tarjeta" type="text" {...register('tarjeta')} style={{ width: '100%', padding: '0.5rem' }} />
        {errors.tarjeta && <span style={{ color: 'red', fontSize: '0.85rem' }}>{errors.tarjeta.message}</span>}
      </div>

      <div>
        <label htmlFor="cvv" style={{ display: 'block', fontWeight: 'bold' }}>CVV (3 dígitos):</label>
        <input id="cvv" type="text" {...register('cvv')} style={{ width: '100%', padding: '0.5rem' }} />
        {errors.cvv && <span style={{ color: 'red', fontSize: '0.85rem' }}>{errors.cvv.message}</span>}
      </div>

      <button type="submit" disabled={isSubmitting} style={{ padding: '0.75rem', cursor: 'pointer', background: 'green', color: '#fff', border: 'none', borderRadius: '4px' }}>
        {isSubmitting ? 'Procesando cargo bancario...' : 'Pagar Orden de Compra'}
      </button>
    </form>
  );
}
```

---

## 15.3 Ejercicios y Retos Avanzados para el Lector

Para consolidar tu aprendizaje como desarrollador sénior de React, te desafiamos a expandir este proyecto integrador completando los siguientes retos:

1.  **Soporte Offline con Zustand**: Modifica la persistencia de Zustand en `cartStore.ts` para que, si el usuario agrega productos al carrito sin internet, los datos se sincronicen y se ejecute una Server Action de checkout asíncrona automáticamente cuando recupere la conexión de red (*Service Worker synchronization*).
2.  **Rutas Protegidas con Cargando**: Configura una ruta `/checkout` protegida en React Router que verifique si el usuario está autenticado consumiendo un Contexto de seguridad. Utiliza Data Loaders para pre-cargar la dirección por defecto del usuario desde la base de datos de forma paralela.
3.  **Integrar Vitest y MSW**: Escribe una prueba de integración completa en Vitest que simule al usuario buscando un artículo en el listado del catálogo, agregándolo al carrito, y verificando que el total de artículos en el Navbar sube de 0 a 1.

---

## Resumen del Capítulo y Cierre del Libro

*   La **Arquitectura Feature-Driven** organiza tu código por funcionalidades y lógica de negocio, desacoplando responsabilidades de forma idéntica a un **rascacielos modular completo** e incrementando la productividad en equipos de desarrollo ágiles.
*   Hemos consolidado la unión de **Zustand para estados globales**, **useTransition para concurrencia fluida**, **React Router para SPA**, y **React Hook Form + Zod para formularios seguros e inmutables** tipados en TypeScript.

¡Felicitaciones! Has completado el libro **React: Guía Completa de Programación Moderna**. Has transitado desde los cimientos elementales del renderizado e inmutabilidad, pasando por el cableado de inyectores y estados, hasta las optimizaciones a gran escala y la arquitectura fullstack de vanguardia de React v19. Estás plenamente preparado para construir, testear y liderar el desarrollo de aplicaciones de nivel enterprise listas para producción.

---

[Capítulo anterior](14-testing.md) | [Inicio](README.md)
