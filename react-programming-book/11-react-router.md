# Capítulo 11: Enrutamiento Moderno con React Router

> "Una ruta en el desarrollo frontend no es simplemente una dirección URL del navegador; es el descriptor de estado más poderoso de tu aplicación, representable de forma jerárquica y modular."

En el desarrollo web tradicional, cambiar de página obligaba al navegador a desechar toda la memoria de la pestaña actual, realizar una petición de red completa por un nuevo archivo HTML, y repintar la pantalla completa desde cero. En las **Single Page Applications (SPA)** modernas, React cambia este comportamiento al implementar el **enrutamiento del lado del cliente (Client-Side Routing)**: el navegador intercepta el cambio de URL y React reemplaza quirúrgicamente en pantalla los componentes pertinentes sin recargar el navegador.

En este capítulo, estudiaremos cómo modelar enrutamientos complejos utilizando **React Router v6+** (el motor oficial del ecosistema), comprenderemos el concepto de **rutas anidadas (Nested Routing)** y aprenderemos a usar **Data Loaders** para erradicar las molestas peticiones en cascada (*waterfalls*).

---

## 11.1 El Enrutamiento del Lado del Cliente (Client-Side Routing)

El enrutamiento en una SPA se basa en sincronizar la barra de direcciones del navegador con el estado visual del árbol de componentes de React. Cuando el usuario hace clic en un enlace:
1.  React Router intercepta el clic y previene la recarga física por defecto del navegador (`e.preventDefault()`).
2.  Actualiza la barra de direcciones URL del navegador utilizando la API **`history.pushState`** de HTML5.
3.  Compara la nueva URL con tu árbol de configuración de rutas, desmonta el componente anterior, y monta el nuevo componente en el área de contenido sin parpadeos de pantalla.

---

## 11.2 Rutas Anidadas: El Concepto de Outlets y Layouts

En interfaces modernas (como dashboards de administración o aplicaciones SaaS), gran parte de la UI permanece estática entre pantallas. El menú superior, la barra lateral de navegación y el pie de página no cambian; solo cambia el cuadro de contenido del centro.

React Router implementa esto de forma brillante mediante **Rutas Anidadas (Nested Routing)** y el elemento contenedor **`<Outlet />`**.

```
┌──────────────────────────────────────────────────────────┐
│  Layout Dashboard (Static: Sidebar, Header)              │
│  ┌────────────────────────────────────────────────────┐  │
│  │  <Outlet />  (Dynamic Content Area)                 │  │
│  │                                                    │  │
│  │  - Si URL es /dashboard/perfil ──► <Perfil />       │  │
│  │  - Si URL es /dashboard/ventas ──► <Ventas />       │  │
│  └────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────┘
```

> [!NOTE]
> ### 🚂 La Analogía de la Estación de Tren Modular
> 
> Para entender las Rutas Anidadas y los Outlets, imagina una gran estación de tren metropolitana modular:
> 
> - El edificio principal de la estación, las ventanillas de venta de billetes, los controles de seguridad y la cafetería de la entrada se mantienen completamente estables y nunca cambian de posición de un día para otro (**El Layout Estático General**).
> - Sin embargo, en el centro de la estación hay andenes de paso dinámicos (**Los `<Outlet />`**).
> - Si cambias de destino en tu billete de tren (la URL), no demueles ni reconstruyes el edificio de la estación de tren desde cero. Los pasajeros simplemente caminan hacia una vía diferente (sub-ruta).
> - React Router encaja el tren de tu andén de destino (**el componente hijo `<Perfil />` o `<Ventas />`**) directamente en el espacio de paso del andén (**el `<Outlet />`**) manteniendo toda la infraestructura exterior de la estación intacta y en funcionamiento.

---

## 11.3 Configuración Declarativa Modernas con Data Loaders

En el React Router moderno (v6+), se promueve abandonar la declaración histórica basada en componentes JSX (`<Routes>`, `<Route>`) a favor de la configuración declarativa mediante objetos nativos de JavaScript usando **`createBrowserRouter`**.

Esta aproximación permite definir **Loaders**: funciones asíncronas que se ejecutan **en paralelo con la descarga del componente y la comparación de la ruta**, descargando la información de la API *antes* de que los componentes de la interfaz comiencen a renderizarse, eliminando por completo las pantallas con spinners intermedios incómodos y las peticiones lentas en cascada (*request waterfalls*).

### Ejemplo Profesional de Rutas y Data Loaders:

#### [router.tsx](file:///Users/andres/Documents/biblioteca/react-programming-book/src/router.tsx)
```typescript
import React from 'react';
import { createBrowserRouter, RouterProvider, Outlet, useLoaderData, useParams, Link } from 'react-router-dom';

// 1. Definir una interfaz de los datos
interface Articulo {
  id: number;
  title: string;
  body: string;
}

// 2. Componente de Layout General Estático
function LayoutPrincipal(): React.JSX.Element {
  return (
    <div style={{ display: 'flex', minHeight: '100vh', fontFamily: 'sans-serif' }}>
      <aside style={{ width: '200px', background: '#f0f0f0', padding: '1rem', borderRight: '1px solid #ccc' }}>
        <h3>Navegación</h3>
        <nav style={{ display: 'flex', flexDirection: 'column', gap: '0.5rem' }}>
          <Link to="/">Inicio</Link>
          <Link to="/articulos">Ver Artículos</Link>
        </nav>
      </aside>
      
      <main style={{ flex: 1, padding: '2rem' }}>
        {/* El andén dinámico de tren donde encajarán las sub-rutas */}
        <Outlet />
      </main>
    </div>
  );
}

// 3. Loader asíncrono para descargar datos en paralelo al ruteo
export async function articulosLoader(): Promise<Articulo[]> {
  const res = await fetch('https://jsonplaceholder.typicode.com/posts?_limit=5');
  if (!res.ok) throw new Error("Error de red");
  return res.json();
}

// 4. Componente de Sub-Ruta que consume el Loader
function ListaArticulos(): React.JSX.Element {
  // useLoaderData extrae automáticamente los datos descargados por el loader
  const articulos = useLoaderData() as Articulo[];

  return (
    <div>
      <h2>Artículos de Tecnología</h2>
      <ul>
        {articulos.map(art => (
          <li key={art.id} style={{ margin: '1rem 0' }}>
            <Link to={`/articulos/${art.id}`} style={{ fontWeight: 'bold', fontSize: '1.1rem' }}>
              {art.title}
            </Link>
          </li>
        ))}
      </ul>
    </div>
  );
}

// 5. Configurar el Router de forma declarativa con objetos
export const router = createBrowserRouter([
  {
    path: '/',
    element: <LayoutPrincipal />, // Layout Padre común
    children: [
      {
        index: true,
        element: <h2>Página de Inicio de la Biblioteca</h2>
      },
      {
        path: 'articulos',
        element: <ListaArticulos />,
        loader: articulosLoader // El loader se gatilla al hacer clic en el Link
      }
    ]
  }
]);
```

---

## Resumen del Capítulo

*   El **enrutamiento del lado del cliente (Client-Side Routing)** permite transicionar pantallas en SPAs de forma instantánea manipulando el historial del navegador sin realizar recargas físicas de HTML.
*   Las **Rutas Anidadas** simplifican el diseño de UI modulares y dashboards. El componente estático actúa como plantilla y los cambios dinámicos se inyectan en el componente contenedor **`<Outlet />`**.
*   Los **Data Loaders** precargan la información de las APIs en paralelo a la validación de la ruta, eliminando las peticiones desorganizadas en cascada (*waterfalls*) y mejorando de forma notable la velocidad percibida.
*   El uso de **`Link`** en lugar de etiquetas de anclaje nativas (`<a>`) es indispensable para que React Router intercepte las peticiones y mantenga el control del historial del navegador.

En el próximo capítulo (iniciando el Bloque de Producción y Formularios), exploraremos cómo construir interfaces interactivas y formularios de alto rendimiento y fácil mantenibilidad implementando **React Hook Form** y validaciones estrictas de esquemas de negocio con **Zod**.

---

[Capítulo anterior](10-server-components-y-actions.md) | [Inicio](README.md) | [Capítulo siguiente →](12-formularios-y-validaciones.md)
